# 12. Pipeline CI/CD - App View v2.0

**Versão:** 1.0  
**Última Atualização:** Janeiro 2025  
**Responsável:** Arquitetura & DevOps

---

## Sumário

1. [Princípios de Design](#principios-de-design)
2. [Visão Geral do Pipeline](#visao-geral-do-pipeline)
3. [Fase 1: Validação de Pull Request](#fase-1-validacao-de-pull-request)
4. [Fase 2: Deploy Blue/Green](#fase-2-deploy-bluegreen)
5. [Fase 3: Validação Pós-Deploy & Rollback](#fase-3-validacao-pos-deploy-rollback)
6. [Estratégia de Rollback](#estrategia-de-rollback)
7. [GitHub Actions Workflows](#github-actions-workflows)
8. [Configuração Azure App Service](#configuracao-azure-app-service)
9. [Notificações Microsoft Teams](#notificacoes-microsoft-teams)
10. [Ambientes & Branching Strategy](#ambientes-branching-strategy)

---

## Princípios de Design

A arquitetura de CI/CD é construída em torno de princípios fundamentais:

### 1. **Fail Early, Fail Cheap**
- Detectar problemas o mais cedo possível (linting, vulnerabilidades, testes)
- Falhas nas primeiras etapas evitam builds e deploys custosos
- **Exemplo:** Spotless falha em 30s vs. deploy que falha em produção após 10min

### 2. **Confidence Before Deployment**
- Uma mudança não pode ser deployada sem passar por:
  - ✅ Quality gates (Spotless, PMD)
  - ✅ Security gates (OWASP Dependency Check)
  - ✅ Correctness gates (testes unitários + integração)
- **Regra:** PRs bloqueados até ALL checks passarem

### 3. **Confidence After Deployment**
- Validações pré-deploy não garantem 100% de confiabilidade
- Ambientes reais revelam problemas reais
- **Validação pós-deploy obrigatória:**
  - Smoke tests em produção
  - Monitoramento de métricas (error rate, latência)
  - Automatic rollback se métricas degradarem

### 4. **Blast-Radius Reduction**
- Deployments em produção usam **Blue/Green**
- Minimiza impacto se algo der errado
- **Fluxo:** Deploy em STAGING → Validação → Swap instantâneo 100%

### 5. **Everything is Observable**
- Cada etapa produz sinais explícitos:
  - Comentários no PR (status checks)
  - Métricas de deploy (latência, error rate)
  - Notificações Teams (sucesso/falha)
- Engenheiros sempre sabem o estado do sistema

---

## Visão Geral do Pipeline

O pipeline de CI/CD é implementado usando **GitHub Actions** e está dividido em **3 fases lógicas**:

```
┌──────────────────────────────────────────────────────────────────┐
│                    FASE 1: PR VALIDATION                         │
│  (Trigger: PR opened/updated)                                    │
│                                                                  │
│  1. Spotless Check (formatting)                                  │
│  2. Maven Build + Unit Tests                                     │
│  3. Integration Tests (Testcontainers)                           │
│  4. OWASP Dependency Check (vulnerabilities)                     │
│  5. PMD/SpotBugs (code quality)                                  │
│  6. PR Smoke Tests (fast validation)                             │
│  7. Update PR with status                                        │
│                                                                  │
│  ❌ ANY failure → PR blocked                                     │
│  ✅ ALL pass → PR can be merged                                 │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│              FASE 2: DEPLOYMENT BLUE/GREEN                       │
│  (Trigger: Merge to main)                                        │
│                                                                  │
│  1. Build Docker Image                                           │
│  2. Push to Azure Container Registry (ACR)                       │
│  3. Deploy to Azure App Service (STAGING slot)                   │
│  4. Run E2E tests against STAGING                                │
│  5. Swap STAGING → PRODUCTION (instantâneo 100%)                 │
│  6. Collect metrics (error rate, latency, throughput)            │
└──────────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────────┐
│         FASE 3: POST-DEPLOY VALIDATION & ROLLBACK                │
│  (Trigger: Automatic after deployment)                           │
│                                                                  │
│  1. Post-Deploy Smoke Tests (production-safe)                    │
│  2. Metrics Evaluation Gate:                                     │
│     - Error rate < 1%                                            │
│     - P95 latency < 2s                                           │
│     - No circuit breakers OPEN                                   │
│  3. Decision Point:                                              │
│     ✅ Metrics healthy → 100% traffic to GREEN                   │
│     ❌ Metrics degraded → AUTOMATIC ROLLBACK                     │
│  4. Notify Teams (success/failure)                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## Fase 1: Validação de Pull Request

### Objetivos
- Garantir qualidade de código
- Prevenir regressões
- Detectar vulnerabilidades de segurança
- Garantir que a aplicação builda e inicia

### Workflow: `pr-validation.yml`

```yaml
name: PR Validation

on:
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]

jobs:
  # Step 1: Code Formatting
  spotless:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run Spotless Check
        run: mvn spotless:check
      
      - name: Comment PR (if failed)
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ **Spotless Check Failed**\n\nRun `mvn spotless:apply` locally to fix formatting.'
            })

  # Step 2: Build + Unit Tests
  build-and-test:
    runs-on: ubuntu-latest
    needs: spotless
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build with Maven
        run: mvn clean package -DskipTests
      
      - name: Run Unit Tests
        run: mvn test -Dtest="**/*UnitTest"
      
      - name: Upload Test Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: target/surefire-reports/

  # Step 3: Integration Tests
  integration-tests:
    runs-on: ubuntu-latest
    needs: build-and-test
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: appview_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run Integration Tests
        run: mvn test -Dtest="**/*IntegrationTest"
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/appview_test
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

  # Step 4: Dependency Vulnerability Scan
  dependency-check:
    runs-on: ubuntu-latest
    needs: spotless
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: OWASP Dependency Check
        run: mvn org.owasp:dependency-check-maven:check
      
      - name: Upload Dependency Check Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: target/dependency-check-report.html

  # Step 5: Code Quality Analysis
  code-quality:
    runs-on: ubuntu-latest
    needs: spotless
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run PMD
        run: mvn pmd:check
      
      - name: Run SpotBugs
        run: mvn spotbugs:check

  # Step 6: PR Smoke Tests
  pr-smoke-tests:
    runs-on: ubuntu-latest
    needs: [build-and-test, integration-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Start Application (Background)
        run: |
          mvn spring-boot:run -Dspring-boot.run.profiles=test &
          sleep 30  # Wait for app to start
      
      - name: Run Smoke Tests
        run: |
          curl -f http://localhost:8080/actuator/health || exit 1
          curl -f http://localhost:8080/actuator/info || exit 1

  # Step 7: PR Summary
  pr-summary:
    runs-on: ubuntu-latest
    needs: [spotless, build-and-test, integration-tests, dependency-check, code-quality, pr-smoke-tests]
    if: always()
    steps:
      - uses: actions/github-script@v7
        with:
          script: |
            const results = {
              spotless: '${{ needs.spotless.result }}',
              build: '${{ needs.build-and-test.result }}',
              integration: '${{ needs.integration-tests.result }}',
              security: '${{ needs.dependency-check.result }}',
              quality: '${{ needs.code-quality.result }}',
              smoke: '${{ needs.pr-smoke-tests.result }}'
            };
            
            const allPassed = Object.values(results).every(r => r === 'success');
            const emoji = allPassed ? '✅' : '❌';
            
            const body = `
            ## ${emoji} PR Validation Summary
            
            | Check | Status |
            |-------|--------|
            | Code Formatting (Spotless) | ${results.spotless === 'success' ? '✅' : '❌'} |
            | Build + Unit Tests | ${results.build === 'success' ? '✅' : '❌'} |
            | Integration Tests | ${results.integration === 'success' ? '✅' : '❌'} |
            | Security Scan (OWASP) | ${results.security === 'success' ? '✅' : '❌'} |
            | Code Quality (PMD/SpotBugs) | ${results.quality === 'success' ? '✅' : '❌'} |
            | Smoke Tests | ${results.smoke === 'success' ? '✅' : '❌'} |
            
            ${allPassed ? '**Ready to merge!** 🚀' : '**Blocked:** Fix failing checks before merging.'}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
```

### Resultado

- ❌ **Qualquer etapa crítica falhar** → PR **bloqueado** (branch protection rules)
- ✅ **Todas as etapas passarem** → PR pode ser mergeado

---

## Fase 2: Deploy Blue/Green

### Objetivos
- Fazer deploy seguro para produção
- Validar em STAGING antes de swap
- Swap instantâneo com rollback rápido

### Workflow: `deploy-production.yml`

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]
  # Gates automáticos verificados no início do workflow

jobs:
  # Step 0: Deployment Gates (validações automáticas)
  deployment-gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check Deployment Window
        run: |
          # Verificar se está em horário de deploy (Ter/Qui 20h-22h)
          DAY=$(date +%u)  # 1=Mon, 2=Tue, ..., 5=Fri
          HOUR=$(date +%H)
          
          if [[ ($DAY -eq 2 || $DAY -eq 4) && $HOUR -ge 20 && $HOUR -lt 22 ]]; then
            echo "✅ Janela de deploy OK (Ter/Qui 20h-22h)"
          else
            echo "❌ Fora da janela de deploy permitida"
            echo "Deploy apenas Terça ou Quinta, 20h-22h"
            exit 1
          fi
      
      - name: Check UAT Validation
        run: |
          # Verificar se PR tem label "uat-validated"
          PR_NUMBER=$(jq -r .pull_request.number "$GITHUB_EVENT_PATH")
          LABELS=$(gh pr view $PR_NUMBER --json labels -q '.labels[].name')
          
          if echo "$LABELS" | grep -q "uat-validated"; then
            echo "✅ UAT validado pelos times de negócio"
          else
            echo "❌ PR não tem label 'uat-validated'"
            echo "Times de negócio devem validar em UAT antes de PRD"
            exit 1
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Check GMUD
        run: |
          # Verificar se commit tem tag "gmud-XXXX" ou annotation
          COMMIT_MESSAGE=$(git log -1 --pretty=%B)
          
          if echo "$COMMIT_MESSAGE" | grep -qE "GMUD-[0-9]+"; then
            GMUD_ID=$(echo "$COMMIT_MESSAGE" | grep -oE "GMUD-[0-9]+")
            echo "✅ GMUD identificada: $GMUD_ID"
          else
            echo "⚠️ AVISO: Nenhuma GMUD identificada no commit"
            echo "Recomendável incluir ID da GMUD no commit message"
            # Não bloqueia, apenas avisa
          fi

  # Step 1: Build Docker Image
  build-image:
    needs: deployment-gates
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build JAR
        run: mvn clean package -DskipTests
      
      - name: Build Docker Image
        run: |
          docker build -t appview:${{ github.sha }} .
          docker tag appview:${{ github.sha }} appview:latest
      
      - name: Login to Azure Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      
      - name: Push to ACR
        run: |
          docker tag appview:${{ github.sha }} ${{ secrets.ACR_LOGIN_SERVER }}/appview:${{ github.sha }}
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/appview:${{ github.sha }}

  # Step 2: Deploy to STAGING slot (Blue/Green)
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-image
    steps:
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy to STAGING Slot
        uses: azure/webapps-deploy@v2
        with:
          app-name: 'appview-prod'
          slot-name: 'staging'
          images: ${{ secrets.ACR_LOGIN_SERVER }}/appview:${{ github.sha }}
      
      - name: Wait for Deployment
        run: sleep 60

  # Step 3: E2E Tests on STAGING
  e2e-tests-staging:
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Run E2E Tests
        run: |
          mvn test -Dtest="**/*E2ETest" \
            -Dtest.base.url=https://appview-prod-staging.azurewebsites.net
      
      - name: Validate Health Endpoints
        run: |
          curl -f https://appview-prod-staging.azurewebsites.net/actuator/health || exit 1

  # Step 4: Blue/Green Swap
  swap-to-production:
    runs-on: ubuntu-latest
    needs: e2e-tests-staging
    steps:
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Swap STAGING → PRODUCTION
        run: |
          az webapp deployment slot swap \
            --name appview-prod \
            --resource-group appview-rg \
            --slot staging \
            --target-slot production
      
      - name: Wait for Slot Swap
        run: sleep 30
```

### Blue/Green Swap

**Não usamos Canary Release!**

**Motivo:** Times de negócio precisam fazer regressão completa imediatamente após deploy.

**Estratégia:**

| Fase | Tráfego GREEN | Duração | Validação |
|------|---------------|---------|----------|
| **Deploy STAGING** | 0% | 5 min | E2E tests, smoke tests |
| **Swap** | 100% | 5 segundos | Instantâneo |
| **Validação** | 100% | Imediata | Times de negócio validam |

**Fluxo:**
1. Deploy para STAGING (GREEN) - 0% tráfego
2. E2E tests em STAGING
3. **Swap instantâneo** - GREEN vira PRD (100% tráfego)
4. Times de negócio fazem regressão completa
5. Se problema: rollback em 5 segundos

---

## Fase 3: Validação Pós-Deploy & Rollback

### Objetivos
- Detectar problemas que só aparecem em produção
- Recuperar automaticamente de deploys falhos

### Workflow: `post-deploy-validation.yml`

```yaml
name: Post-Deploy Validation

on:
  workflow_run:
    workflows: ["Deploy to Production"]
    types: [completed]

jobs:
  # Step 1: Post-Deploy Smoke Tests
  smoke-tests:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Production Smoke Tests
        run: |
          mvn test -Dtest="**/*SmokeTest" \
            -Dtest.base.url=https://appview-prod.azurewebsites.net
      
      - name: Validate Critical Flows
        run: |
          # Health check
          curl -f https://appview-prod.azurewebsites.net/actuator/health || exit 1
          
          # Login flow (production-safe)
          curl -X POST https://appview-prod.azurewebsites.net/api/auth/health \
            -H "Content-Type: application/json" || exit 1

  # Step 2: Metrics Evaluation
  evaluate-metrics:
    runs-on: ubuntu-latest
    needs: smoke-tests
    steps:
      - name: Fetch Application Insights Metrics
        id: metrics
        run: |
          # Error rate
          ERROR_RATE=$(curl -s "https://api.applicationinsights.io/v1/apps/${{ secrets.APP_INSIGHTS_APP_ID }}/metrics/requests/failed" \
            -H "x-api-key: ${{ secrets.APP_INSIGHTS_API_KEY }}" | jq '.value.rate')
          
          # P95 latency
          P95_LATENCY=$(curl -s "https://api.applicationinsights.io/v1/apps/${{ secrets.APP_INSIGHTS_APP_ID }}/metrics/requests/duration" \
            -H "x-api-key: ${{ secrets.APP_INSIGHTS_API_KEY }}" | jq '.value.p95')
          
          echo "error_rate=$ERROR_RATE" >> $GITHUB_OUTPUT
          echo "p95_latency=$P95_LATENCY" >> $GITHUB_OUTPUT
      
      - name: Evaluate Thresholds
        run: |
          ERROR_RATE=${{ steps.metrics.outputs.error_rate }}
          P95_LATENCY=${{ steps.metrics.outputs.p95_latency }}
          
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "❌ Error rate too high: $ERROR_RATE"
            exit 1
          fi
          
          if (( $(echo "$P95_LATENCY > 2000" | bc -l) )); then
            echo "❌ P95 latency too high: ${P95_LATENCY}ms"
            exit 1
          fi
          
          echo "✅ Metrics healthy: Error rate $ERROR_RATE, P95 ${P95_LATENCY}ms"

  # Step 3: Automatic Rollback on Failure
  rollback:
    runs-on: ubuntu-latest
    needs: [smoke-tests, evaluate-metrics]
    if: failure()
    steps:
      - uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Rollback to Previous Version
        run: |
          echo "🚨 Metrics degraded - ROLLING BACK"
          
          az webapp deployment slot swap \
            --name appview-prod \
            --resource-group appview-rg \
            --slot production \
            --target-slot staging
      
      - name: Notify Teams (Rollback)
        uses: aliencube/microsoft-teams-actions@v0.8.0
        with:
          webhook_uri: ${{ secrets.TEAMS_WEBHOOK_URL }}
          title: "🚨 ROLLBACK EXECUTED"
          summary: "Deploy failed validation and was automatically rolled back."
          theme_color: "FF0000"
          sections: |
            [
              {
                "activityTitle": "Deployment Rollback",
                "activitySubtitle": "Commit: ${{ github.sha }}",
                "facts": [
                  { "name": "Reason", "value": "Post-deploy metrics degraded" },
                  { "name": "Action", "value": "Automatic rollback to previous version" },
                  { "name": "Status", "value": "🔴 ROLLED BACK" }
                ]
              }
            ]

  # Step 4: Success Notification
  notify-success:
    runs-on: ubuntu-latest
    needs: [smoke-tests, evaluate-metrics]
    if: success()
    steps:
      - name: Notify Teams (Success)
        uses: aliencube/microsoft-teams-actions@v0.8.0
        with:
          webhook_uri: ${{ secrets.TEAMS_WEBHOOK_URL }}
          title: "✅ Deployment Successful"
          summary: "App View v2.0 deployed to production successfully."
          theme_color: "00FF00"
          sections: |
            [
              {
                "activityTitle": "Production Deployment",
                "activitySubtitle": "Commit: ${{ github.sha }}",
                "facts": [
                  { "name": "Environment", "value": "Production" },
                  { "name": "Version", "value": "${{ github.sha }}" },
                  { "name": "Author", "value": "${{ github.actor }}" },
                  { "name": "Status", "value": "🟢 DEPLOYED" }
                ],
                "markdown": true
              }
            ]
```

### Métricas de Avaliação

| Métrica | Threshold | Ação se Exceder |
|---------|-----------|-----------------|
| **Error Rate** | < 1% | Rollback automático |
| **P95 Latency** | < 2s | Rollback automático |
| **Circuit Breaker OPEN** | 0 | Rollback automático |
| **Smoke Tests** | 100% pass | Rollback automático |

---

## Estratégia de Rollback

### Por que Rollback é Critical

- Deployments podem falhar de formas imprevisíveis em produção
- Tempo de recuperação deve ser **< 5 minutos**
- Usuários não devem notar falhas

### Como Funciona

1. **Blue/Green Slots**: Versão anterior (BLUE) continua rodando
2. **Sem rebuild**: Rollback é simplesmente um swap de slots
3. **Instantâneo**: Azure App Service swap leva ~5 segundos
4. **Automatic**: Não requer intervenção manual

### Exemplo de Rollback

```bash
# Deploy falha após 100% traffic shift
# Sistema detecta error rate > 1%

az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot production \
  --target-slot staging

# Em 5 segundos, sistema volta para versão anterior
# Tráfego 100% de volta para BLUE (versão estável)
```

### Manual Rollback

Se necessário, operador pode fazer rollback manual:

```bash
# Via Azure CLI
az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot production \
  --target-slot staging

# Ou via Portal Azure
# App Service → appview-prod → Deployment slots → Swap
```

---

## Configuração Azure App Service

### Deployment Slots

```bash
# Criar slot STAGING
az webapp deployment slot create \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging

# Configurar slot settings (não swappeados)
az webapp config appsettings set \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging \
  --slot-settings \
    SPRING_PROFILES_ACTIVE=staging \
    APPLICATIONINSIGHTS_CONNECTION_STRING=<staging-connection-string>
```

### Traffic Routing

```bash
# Configurar canary (10% para staging)
az webapp traffic-routing set \
  --name appview-prod \
  --resource-group appview-rg \
  --distribution staging=10

# Limpar routing (100% para production)
az webapp traffic-routing clear \
  --name appview-prod \
  --resource-group appview-rg
```

### Container Registry

```bash
# Criar Azure Container Registry
az acr create \
  --name appviewacr \
  --resource-group appview-rg \
  --sku Standard

# Configurar App Service para usar ACR
az webapp config container set \
  --name appview-prod \
  --resource-group appview-rg \
  --docker-custom-image-name appviewacr.azurecr.io/appview:latest \
  --docker-registry-server-url https://appviewacr.azurecr.io \
  --docker-registry-server-user <username> \
  --docker-registry-server-password <password>
```

---

## Notificações Microsoft Teams

### Configurar Webhook

1. Acessar canal Teams onde notificações devem chegar
2. Clicar em `...` → `Connectors` → `Incoming Webhook`
3. Nomear webhook: `AppView CI/CD`
4. Copiar URL do webhook
5. Adicionar como secret no GitHub: `TEAMS_WEBHOOK_URL`

### Tipos de Notificações

#### 1. Deploy Iniciado

```json
{
  "title": "🚀 Deploy Iniciado",
  "summary": "Deploy para produção em andamento",
  "themeColor": "0078D4",
  "sections": [
    {
      "activityTitle": "Production Deployment",
      "activitySubtitle": "Commit: abc123",
      "facts": [
        { "name": "Branch", "value": "main" },
        { "name": "Author", "value": "jhonny" },
        { "name": "Status", "value": "🟡 IN PROGRESS" }
      ]
    }
  ]
}
```

#### 2. Deploy com Sucesso

```json
{
  "title": "✅ Deploy Successful",
  "summary": "App View v2.0 deployed successfully",
  "themeColor": "00FF00",
  "sections": [
    {
      "activityTitle": "Production Deployment",
      "facts": [
        { "name": "Version", "value": "abc123" },
        { "name": "Duration", "value": "12m 34s" },
        { "name": "Status", "value": "🟢 DEPLOYED" }
      ]
    }
  ]
}
```

#### 3. Deploy Falhou / Rollback

```json
{
  "title": "🚨 ROLLBACK EXECUTED",
  "summary": "Deploy failed and was rolled back",
  "themeColor": "FF0000",
  "sections": [
    {
      "activityTitle": "Deployment Rollback",
      "facts": [
        { "name": "Reason", "value": "Error rate > 1%" },
        { "name": "Action", "value": "Automatic rollback" },
        { "name": "Status", "value": "🔴 ROLLED BACK" }
      ]
    }
  ]
}
```

#### 4. PR Merged

```json
{
  "title": "🔀 Pull Request Merged",
  "summary": "PR #123 merged to main",
  "themeColor": "6264A7",
  "sections": [
    {
      "activityTitle": "PR #123: Fix sinistro validation",
      "facts": [
        { "name": "Author", "value": "jhonny" },
        { "name": "Reviewers", "value": "maria, joao" },
        { "name": "Status", "value": "✅ MERGED" }
      ]
    }
  ]
}
```

---

## Ambientes & Branching Strategy

### Ambientes

| Ambiente | Branch | Deploy Trigger | URL |
|----------|--------|----------------|-----|
| **DEV** | `develop` | Push automático | `https://appview-dev.azurewebsites.net` |
| **UAT** | `uat` | Merge automático | `https://appview-uat.azurewebsites.net` |
| **PROD** | `main` | Merge automático (com gates) | `https://appview-prod.azurewebsites.net` |
| **PROD (Staging Slot)** | - | Deploy antes de swap | `https://appview-prod-staging.azurewebsites.net` |

### Branching Strategy (GitFlow)

```
main (production)
  ↑
release/v2.1.0 (UAT)
  ↑
develop (DEV)
  ↑
feature/sinistro-validation
feature/alfresco-retry
hotfix/critical-bug
```

### Regras de Branch Protection

#### `main` (Production)

```yaml
branch_protection:
  required_status_checks:
    - pr-validation / spotless
    - pr-validation / build-and-test
    - pr-validation / integration-tests
    - pr-validation / dependency-check
    - pr-validation / code-quality
    - pr-validation / pr-smoke-tests
  
  required_pull_request_reviews:
    required_approving_review_count: 2
    dismiss_stale_reviews: true
    require_code_owner_reviews: true
  
  enforce_admins: false
  allow_force_pushes: false
  allow_deletions: false
```

#### `develop` (Development)

```yaml
branch_protection:
  required_status_checks:
    - pr-validation / build-and-test
    - pr-validation / integration-tests
  
  required_pull_request_reviews:
    required_approving_review_count: 1
```

---

## Workflows Completos - Resumo

### 1. `pr-validation.yml`

**Trigger:** PR opened/updated  
**Duração:** ~8-10 minutos  
**Jobs:**
- Spotless (30s)
- Build + Unit Tests (3 min)
- Integration Tests (3 min)
- OWASP Dependency Check (2 min)
- PMD/SpotBugs (1 min)
- PR Smoke Tests (30s)
- PR Summary (10s)

### 2. `deploy-production.yml`

**Trigger:** Merge to `main`  
**Duração:** ~15 minutos  
**Jobs:**
- **Deployment Gates** (30s) - Valida janela, UAT, GMUD
- Build Docker Image (5 min)
- Deploy to STAGING Slot (2 min)
- E2E Tests on STAGING (5 min)
- Swap to Production (30s)
- Post-deploy validation (2 min)

### 3. `post-deploy-validation.yml`

**Trigger:** After `deploy-production.yml` completes  
**Duração:** ~3-5 minutos  
**Jobs:**
- Post-Deploy Smoke Tests (2 min)
- Evaluate Metrics (1 min)
- Rollback (if failure) (30s)
- Notify Teams (10s)

---

## Secrets Necessários (GitHub)

### Azure

- `AZURE_CREDENTIALS` - Service Principal JSON
- `ACR_LOGIN_SERVER` - `appviewacr.azurecr.io`
- `ACR_USERNAME` - Azure Container Registry username
- `ACR_PASSWORD` - Azure Container Registry password

### Application Insights

- `APP_INSIGHTS_APP_ID` - Application ID
- `APP_INSIGHTS_API_KEY` - API key para queries

### Microsoft Teams

- `TEAMS_WEBHOOK_URL` - Incoming Webhook URL

### Exemplo de configuração

```bash
# Criar Service Principal para GitHub Actions
az ad sp create-for-rbac \
  --name "github-actions-appview" \
  --role contributor \
  --scopes /subscriptions/<subscription-id>/resourceGroups/appview-rg \
  --sdk-auth

# Output (adicionar como secret AZURE_CREDENTIALS):
{
  "clientId": "xxx",
  "clientSecret": "xxx",
  "subscriptionId": "xxx",
  "tenantId": "xxx"
}
```

---

## Monitoramento do Pipeline

### Azure Application Insights - Queries

#### Error Rate durante Deploy

```kql
requests
| where timestamp > ago(30m)
| summarize 
    total = count(),
    failed = countif(success == false)
| extend error_rate = todouble(failed) / todouble(total)
| project error_rate
```

#### P95 Latency

```kql
requests
| where timestamp > ago(30m)
| summarize percentile(duration, 95)
```

#### Circuit Breakers OPEN

```kql
customEvents
| where timestamp > ago(30m)
| where name == "CircuitBreakerStateChanged"
| where customDimensions.newState == "OPEN"
| summarize count() by tostring(customDimensions.circuitBreakerName)
```

### GitHub Actions Dashboard

Criar dashboard customizado no GitHub:

```yaml
# .github/workflows/dashboard.yml
name: CI/CD Dashboard

on:
  schedule:
    - cron: '0 * * * *'  # Every hour

jobs:
  update-dashboard:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch Workflow Runs
        run: |
          gh api repos/${{ github.repository }}/actions/runs \
            --jq '.workflow_runs[] | {name: .name, status: .status, conclusion: .conclusion}'
```

---

## Troubleshooting

### Pipeline falha em "Spotless Check"

**Problema:** Código não está formatado corretamente

**Solução:**
```bash
mvn spotless:apply
git add .
git commit --amend --no-edit
git push --force
```

### Pipeline falha em "Integration Tests"

**Problema:** Testcontainers não consegue iniciar PostgreSQL

**Solução:**
- Verificar se Docker está rodando no runner
- Aumentar timeout: `TESTCONTAINERS_RYUK_DISABLED=true`

### Deploy falha em "E2E Tests on STAGING"

**Problema:** Testes E2E falhando em STAGING

**Solução:**
- Investigar logs em STAGING slot
- Verificar se algum sistema externo está down
- NÃO fazer swap enquanto STAGING não estiver OK
- Corrigir problema e fazer novo deploy para STAGING

### Rollback manual necessário

**Problema:** Deploy passou por todas validações mas problemas aparecem depois

**Solução:**
```bash
# Rollback via CLI
az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot production \
  --target-slot staging

# Investigar causa raiz
az monitor app-insights query \
  --app <app-id> \
  --analytics-query "exceptions | where timestamp > ago(1h)"
```

---

## Comparação com Outras Estratégias

### CI/CD App View v2.0 vs. Alternativas

| Aspecto | Nossa Estratégia | Rolling Deployment | Recreate | Kubernetes Blue/Green |
|---------|------------------|---------------------|----------|------------------------|
| **Tempo de Rollback** | 5s | 5-10min | 10-15min | 30s-1min |
| **Downtime** | Zero | Zero | 5-10min | Zero |
| **Complexidade** | Baixa | Média | Baixa | Alta |
| **Custo** | Médio | Baixo | Baixo | Alto |
| **Blast Radius** | 0-100% (instant) | 100% gradual | 100% | 0-100% |
| **Ideal para** | Small team, critical system | High-traffic, stateless | Non-critical, low SLA | Large team, complex infra |

---

## Por que Esta Arquitetura?

Esta arquitetura de CI/CD foi escolhida porque:

1. ✅ **Funciona para times pequenos** - Não requer expertise em Kubernetes
2. ✅ **Escala para confiabilidade enterprise** - Blue/Green com swap instantâneo
3. ✅ **Minimiza riscos em sistemas críticos** - Rollback em 5 segundos
4. ✅ **Encoraja deploys frequentes e seguros** - Validação em cada etapa
5. ✅ **Torna falhas visíveis e recuperáveis** - Notificações Teams + métricas
6. ✅ **Validação de negócio imediata** - Times validam após swap, sem esperar Canary

Balanceia **velocidade**, **segurança** e **clareza**, garantindo que cada deploy é intencional, observável e reversível.

---

## Checklist de Implementação

### Fase 1: Setup Inicial

- [ ] Criar Azure Container Registry
- [ ] Configurar deployment slots (staging)
- [ ] Criar Service Principal para GitHub Actions
- [ ] Configurar secrets no GitHub
- [ ] Criar Incoming Webhook no Teams

### Fase 2: Workflows

- [ ] Implementar `pr-validation.yml`
- [ ] Implementar `deploy-production.yml`
- [ ] Implementar `post-deploy-validation.yml`
- [ ] Configurar branch protection rules

### Fase 3: Validação

- [ ] Testar PR validation com PR de teste
- [ ] Testar deploy to staging
- [ ] Testar blue/green swap instantâneo
- [ ] Testar rollback automático
- [ ] Validar notificações Teams
- [ ] Validar regressão manual pós-deploy com times

### Fase 4: Monitoramento

- [ ] Configurar dashboards no Application Insights
- [ ] Configurar alertas para métricas críticas
- [ ] Documentar runbooks para troubleshooting

---

**Fim do Documento**
