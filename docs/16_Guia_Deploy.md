# 16. Guia de Deploy - App View v2.0

**Versão:** 1.0  
**Última Atualização:** Janeiro 2025  
**Responsável:** Arquitetura & DevOps

---

## Sumário

1. [Visão Geral](#visao-geral)
2. [Ambientes](#ambientes)
3. [Estratégia de Deploy](#estrategia-de-deploy)
4. [Fluxo de Deploy por Ambiente](#fluxo-de-deploy-por-ambiente)
5. [Blue/Green Deployment](#bluegreen-deployment)
6. [Rollback](#rollback)
7. [Checklist de Deploy](#checklist-de-deploy)
8. [GMUD](#gmud)
9. [Troubleshooting](#troubleshooting)

---

## Visão Geral

O processo de deploy do App View v2.0 segue uma abordagem **simples e confiável** ("feijão com arroz"), priorizando:

- ✅ **Validação manual** com times de negócio (em UAT)
- ✅ **Blue/Green deployment** para zero downtime
- ✅ **Rollback instantâneo** em caso de problemas
- ✅ **GMUD controlada** para mudanças em produção
- ✅ **CI/CD 100% automático** (DEV → UAT → PRD)

### Princípios

1. **Fail-Safe:** Sempre ter versão anterior rodando para rollback
2. **Validação Humana:** Times de negócio validam em UAT antes de PRD
3. **Zero Downtime:** Usuários não percebem o deploy
4. **Rastreabilidade:** Cada deploy tem ID, versão e responsável

---

## Ambientes

### Estrutura de Ambientes

```
┌─────────────────────────────────────────────────────┐
│                       DEV                           │
│  - Desenvolvimento contínuo                         │
│  - Deploy automático (push to dev branch)           │
│  - Sem validação de negócio                         │
│  - https://appview-dev.azurewebsites.net            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                       UAT                           │
│  - Homologação / pré-produção                       │
│  - Deploy automático (merge to uat branch)          │
│  - ✅ VALIDAÇÃO OBRIGATÓRIA com negócio             │
│  - https://appview-uat.azurewebsites.net            │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                       PRD                           │
│  - Produção                                         │
│  - Deploy manual (aprovação + GMUD aberta)          │
│  - Blue/Green deployment                            │
│  - https://appview-prod.azurewebsites.net           │
└─────────────────────────────────────────────────────┘
```

### Configurações por Ambiente

| Ambiente | URL | Recursos | Dados | Deploy | Validação |
|----------|-----|----------|-------|--------|-----------|
| **DEV** | appview-dev | B2 (1 instância) | Sintéticos | Automático | Técnica |
| **UAT** | appview-uat | S1 (2 instâncias) | Cópia PRD (anonimizado) | Automático | **Negócio** |
| **PRD** | appview-prod | P2v3 (2+ instâncias) | Real | **Automático (com gates)** | Negócio + GMUD |

---

## Estratégia de Deploy

### Resumo

**Não usamos Canary Release!**

- **Motivo:** Times de negócio precisam fazer **regressão completa** imediatamente após deploy
- **Solução:** Blue/Green deployment com swap instantâneo
- **Validação:** Em UAT antes de PRD, não em produção gradualmente

### Blue/Green Deployment

Estratégia onde **duas versões** do sistema rodam simultaneamente:

- **BLUE (Atual):** Versão estável em produção
- **GREEN (Nova):** Versão sendo deployada

**Fluxo:**
1. Deploy GREEN (nova versão) enquanto BLUE está ativa
2. Testar GREEN (smoke tests automáticos)
3. **Swap instantâneo:** GREEN vira produção, BLUE vira staging
4. Times de negócio fazem regressão completa
5. Se tudo OK, GREEN permanece; se houver problema, **rollback para BLUE em 5 segundos**

```
ANTES DO DEPLOY:
┌─────────────┐
│ BLUE (PRD)  │ ← 100% tráfego
└─────────────┘
┌─────────────┐
│ STAGING     │ ← vazio
└─────────────┘

DURANTE O DEPLOY:
┌─────────────┐
│ BLUE (PRD)  │ ← 100% tráfego (ainda)
└─────────────┘
┌─────────────┐
│ GREEN (STG) │ ← nova versão deployada, 0% tráfego
└─────────────┘

APÓS SWAP:
┌─────────────┐
│ BLUE (STG)  │ ← versão antiga (backup)
└─────────────┘
┌─────────────┐
│ GREEN (PRD) │ ← 100% tráfego, nova versão
└─────────────┘

SE DER PROBLEMA:
(rollback = swap de volta)
┌─────────────┐
│ BLUE (PRD)  │ ← 100% tráfego (voltou)
└─────────────┘
┌─────────────┐
│ GREEN (STG) │ ← versão com problema
└─────────────┘
```

---

## Fluxo de Deploy por Ambiente

### 1. Deploy para DEV

**Trigger:** Push automático para branch `dev`

**Fluxo Automático:**
```bash
1. Developer faz merge de PR → dev
2. GitHub Actions inicia pipeline
3. Build → Testes → Docker Image
4. Push para Azure Container Registry
5. Deploy para appview-dev
6. Smoke tests automáticos
7. ✅ Deploy concluído

Duração: ~8 minutos
```

**Validação:**
- ✅ Testes automatizados passam
- ✅ Smoke tests (health checks)
- ❌ Sem validação de negócio

**Notificação:** Teams (canal #dev-deploys)

---

### 2. Deploy para UAT

**Trigger:** Merge de `dev` → `uat` (após features prontas)

**Pré-requisitos:**
- [ ] Features testadas em DEV
- [ ] Times de negócio avisados
- [ ] Regressão planejada

**Fluxo Automático:**
```bash
1. Tech Lead merge dev → uat
2. GitHub Actions inicia pipeline
3. Build → Testes → Docker Image
4. Push para Azure Container Registry
5. Deploy para appview-uat
6. E2E tests automáticos
7. ✅ Deploy concluído

Duração: ~12 minutos
```

**Validação Manual:**

✅ **OBRIGATÓRIA** - Times de negócio devem validar:

| Time | O que Validar | Prazo |
|------|---------------|-------|
| **Operações Sinistro** | Criar sinistro, anexar docs, OCR | 2 dias |
| **Jurídico** | Portal PJ, perícias, relatórios | 2 dias |
| **Gestão** | Dashboards, métricas, auditoria | 1 dia |

**Checklist de Validação UAT:**

- [ ] ✅ Criar sinistro completo (fluxo feliz)
- [ ] ✅ Anexar documento + OCR
- [ ] ✅ Consultar sinistros (filtros, paginação)
- [ ] ✅ Agendar perícia
- [ ] ✅ Dashboard de métricas
- [ ] ✅ Relatórios (exportação Excel/PDF)
- [ ] ✅ Autenticação e permissões (RBAC)
- [ ] ✅ Casos de borda (CPF inválido, etc.)

**Aprovação:**
- ✅ **TODOS os times** devem aprovar antes de ir para PRD
- ✅ Registro em Jira/Confluence

**Notificação:** Teams (canal #uat-deploys + @times-negocio)

---

### 3. Deploy para PRD

**Trigger:** Automático via GitHub Actions (após UAT validado + GMUD aberta)

**Pré-requisitos:**

✅ **Checklist Pré-Deploy:**

- [ ] ✅ Validação UAT concluída (todos os times)
- [ ] ✅ GMUD aberta e aprovada
- [ ] ✅ Janela de deploy confirmada
- [ ] ✅ Backup do banco realizado
- [ ] ✅ Times de suporte avisados
- [ ] ✅ Comunicado enviado (usuários finais)
- [ ] ✅ Rollback plan documentado
- [ ] ✅ 2 aprovadores (Tech Lead + Gerente)

**Fluxo Totalmente Automático:**

```bash
1. Merge UAT → MAIN (via PR aprovado)
2. GitHub Actions detecta merge
3. Pipeline checa gates automáticos:
   - ✅ GMUD aberta? (API check ou tag)
   - ✅ UAT validado? (label no PR)
   - ✅ Horário de janela OK? (20h-22h Ter/Qui)
4. Se TODOS os gates passarem:
5. Pipeline automatizado continua:
   a. Build → Testes → Docker Image
   b. Push para Azure Container Registry
   c. Deploy para STAGING slot (GREEN)
   d. E2E tests em STAGING
   e. Se E2E passou → continua automaticamente
6. Blue/Green Swap (GREEN → PRD)
7. Smoke tests automáticos em PRD
8. ✅ Deploy concluído

Duração: ~15 minutos
```

**Validação Pós-Deploy:**

✅ **Imediata (automática):**
- Health checks (30s)
- Smoke tests (2 min)
- Métricas (error rate, latência)

✅ **Manual (times de negócio):**

**IMPORTANTE:** Times devem fazer **regressão completa** IMEDIATAMENTE:

| Time | Ação | Prazo |
|------|------|-------|
| **Operações** | Regressão completa de sinistros | 1 hora |
| **Jurídico** | Validação Portal PJ | 1 hora |
| **Suporte** | Validação help desk | 30 min |

**Checklist Regressão PRD:**
- [ ] ✅ Criar sinistro (caso real)
- [ ] ✅ Consultar sinistros existentes
- [ ] ✅ Anexar documento real
- [ ] ✅ Dashboard de métricas
- [ ] ✅ Relatórios
- [ ] ✅ Integrações externas (Alfresco, Jarvis, Pega)

**Notificação:** 
- Teams (canal #prod-deploys + @everyone)
- Email para stakeholders

**Janelas de Deploy:**
- **Preferencial:** Terça ou Quinta, 20h-22h
- **Proibido:** Sexta após 18h, Sábado/Domingo

---

## Blue/Green Deployment

### Configuração Azure App Service

#### Criar Slot STAGING

```bash
# Via Azure CLI
az webapp deployment slot create \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging

# Configurar slot com settings específicos (não swappeados)
az webapp config appsettings set \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging \
  --slot-settings \
    SPRING_PROFILES_ACTIVE=staging \
    APPLICATIONINSIGHTS_CONNECTION_STRING=$STAGING_APP_INSIGHTS
```

### Deploy para STAGING (GREEN)

```bash
# Deploy nova versão para slot staging
az webapp deployment source config-zip \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging \
  --src appview-v2.1.0.zip
```

### Testar STAGING

```bash
# URL staging
https://appview-prod-staging.azurewebsites.net

# Health check
curl https://appview-prod-staging.azurewebsites.net/actuator/health

# Smoke tests
mvn test -Dtest="SmokeTest" \
  -Dtest.base.url=https://appview-prod-staging.azurewebsites.net
```

### Swap para PRODUCTION

```bash
# Swap instantâneo (5 segundos)
az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot staging \
  --target-slot production

# Verificar swap
az webapp deployment slot list \
  --name appview-prod \
  --resource-group appview-rg
```

### Validar Production

```bash
# Health check produção
curl https://appview-prod.azurewebsites.net/actuator/health

# Verificar logs
az webapp log tail \
  --name appview-prod \
  --resource-group appview-rg

# Verificar métricas Application Insights
# (via Portal Azure ou API)
```

---

## Rollback

### Quando Fazer Rollback?

🚨 **Rollback IMEDIATO se:**
- Error rate > 5%
- P95 latency > 5s
- Health checks falhando
- Integração externa quebrada (Alfresco, Jarvis, Pega)
- **Times de negócio reportam problema crítico**

⚠️ **Avaliar rollback se:**
- Error rate 1-5%
- Bug não crítico reportado
- Performance degradada (não crítica)

### Executar Rollback

#### Via Azure Portal (UI)

1. Acessar Azure Portal
2. App Service → `appview-prod`
3. Deployment slots → **Swap**
4. Swap: `production` ↔ `staging`
5. Confirmar

**Duração:** ~30 segundos (via UI)

#### Via Azure CLI (Recomendado)

```bash
# Rollback = swap de volta
az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot production \
  --target-slot staging

# Verificar rollback
curl https://appview-prod.azurewebsites.net/actuator/info
# (verificar versão voltou)
```

**Duração:** ~5 segundos

### Validar Rollback

```bash
# Health check
curl https://appview-prod.azurewebsites.net/actuator/health

# Verificar logs
az webapp log tail --name appview-prod --resource-group appview-rg

# Verificar métricas
# (error rate deve normalizar em 1-2 minutos)
```

### Comunicar Rollback

✅ **SEMPRE comunicar:**

1. **Imediato:** Teams (canal #prod-deploys)
   ```
   🚨 ROLLBACK EXECUTADO
   
   Motivo: Error rate > 5%
   Ação: Rollback para v2.0.5
   Status: Sistema estável
   Próximos passos: Investigar causa raiz
   ```

2. **30 minutos:** Email stakeholders
3. **4 horas:** Post-mortem agendado
4. **24 horas:** Relatório de incidente

---

## Checklist de Deploy

### Pré-Deploy

#### DEV
- [ ] PR aprovado
- [ ] Testes passam
- [ ] CI/CD green

#### UAT
- [ ] Features testadas em DEV
- [ ] Times de negócio avisados
- [ ] Data de validação agendada

#### PRD
- [ ] ✅ Validação UAT completa (TODOS os times)
- [ ] ✅ GMUD aberta (número: GMUD-XXXX)
- [ ] ✅ Janela de deploy confirmada (data/hora)
- [ ] ✅ Backup banco realizado
- [ ] ✅ Times avisados (operação, suporte, jurídico)
- [ ] ✅ Comunicado enviado (usuários finais)
- [ ] ✅ Rollback plan documentado
- [ ] ✅ 2 aprovadores confirmados

### Durante Deploy

- [ ] Pipeline executa sem erros
- [ ] Build sucesso
- [ ] Testes automáticos passam
- [ ] Docker image criada
- [ ] Deploy para STAGING OK
- [ ] Smoke tests em STAGING OK
- [ ] Aprovação manual obtida
- [ ] Swap executado
- [ ] Health checks produção OK

### Pós-Deploy

#### Imediato (0-30 min)
- [ ] Smoke tests automáticos OK
- [ ] Health checks OK
- [ ] Error rate < 1%
- [ ] Latência normal
- [ ] Logs sem erros críticos

#### Curto Prazo (30 min - 2 horas)
- [ ] Times de negócio validam
- [ ] Regressão completa OK
- [ ] Usuários não reportam problemas
- [ ] Métricas estáveis

#### Longo Prazo (2-24 horas)
- [ ] Monitoramento contínuo
- [ ] Nenhum incidente reportado
- [ ] Performance estável
- [ ] Integrações funcionando

---

## GMUD

### O que é GMUD?

**Gestão de Mudanças** - Processo formal para aprovar mudanças em produção.

### Quando é Necessária?

| Mudança | GMUD? |
|---------|-------|
| **Deploy PRD** | ✅ SIM (sempre) |
| Deploy UAT | ❌ Não |
| Deploy DEV | ❌ Não |
| Hotfix PRD | ✅ SIM (simplificada) |
| Alteração config PRD | ✅ SIM |
| Rollback | ❌ Não (emergencial) |

### Processo GMUD

#### 1. Abertura (3-5 dias antes)

**Responsável:** Tech Lead

**Informações:**
```
Título: Deploy App View v2.1.0 - Nova feature de OCR

Descrição:
- Adiciona integração com Jarvis OCR
- Melhoria performance de consultas
- Bug fix: validação de CPF

Impacto:
- Sistema: App View v2.0
- Módulos afetados: Sinistro, Documento
- Usuários impactados: Todos
- Downtime: Zero (Blue/Green)

Janela de Deploy:
- Data: 30/01/2025
- Horário: 20h00 - 22h00
- Duração: 20 minutos

Testes:
- UAT validado: 25/01/2025
- Testes automatizados: OK
- Aprovação negócio: OK

Rollback:
- Estratégia: Blue/Green swap
- Tempo: 5 segundos
- Responsável: DevOps on-call

Aprovadores:
- Tech Lead: João Silva
- Gerente TI: Maria Santos
```

#### 2. Aprovação (até 1 dia antes)

**Aprovadores:**
- Tech Lead
- Gerente TI
- Gerente Operações (se impacto alto)

**Critérios:**
- ✅ UAT validado
- ✅ Rollback plan claro
- ✅ Times avisados
- ✅ Janela de deploy adequada

#### 3. Execução

**Durante o deploy:**
- Atualizar GMUD com status (iniciado, concluído, problema)
- Registrar horários reais
- Documentar desvios do planejado

#### 4. Fechamento (até 24h depois)

**Informações:**
```
Status: ✅ Concluído com sucesso

Horários:
- Início: 20h05
- Deploy STAGING: 20h15
- Swap: 20h18
- Validação: 20h30
- Conclusão: 20h45

Incidentes: Nenhum

Próximos passos: Monitoramento 48h
```

### Hotfix GMUD (Simplificada)

Para **bugs críticos** em produção:

- **Prazo:** 2 horas (não 3 dias)
- **Aprovação:** 1 aprovador (não 2)
- **Validação:** Mínima (smoke tests apenas)

```
Título: HOTFIX - Corrige cálculo de valor

Severidade: CRÍTICA
Impacto: Cálculo incorreto gerando valores errados

Rollback: Blue/Green swap (5s)

Aprovador: Tech Lead
```

---

## Troubleshooting

### Problema: Deploy falha em "Build"

**Erro:**
```
[ERROR] Failed to execute goal org.springframework.boot:spring-boot-maven-plugin
```

**Causa:** Dependência Maven corrompida ou faltando

**Solução:**
```bash
# Limpar cache Maven
mvn clean
rm -rf ~/.m2/repository/*

# Rebuild
mvn clean install
```

---

### Problema: Deploy falha em "Smoke Tests"

**Erro:**
```
Health check failed: status DOWN
```

**Causa:** Aplicação não iniciou corretamente (config, banco, integração)

**Solução:**
```bash
# Ver logs
az webapp log tail --name appview-prod --slot staging --resource-group appview-rg

# Verificar configs
az webapp config appsettings list --name appview-prod --slot staging --resource-group appview-rg

# Verificar conectividade banco
az postgres flexible-server show --name appview-db --resource-group appview-rg
```

---

### Problema: Swap falha

**Erro:**
```
Swap operation failed: Slot is not ready
```

**Causa:** STAGING não está saudável (health check DOWN)

**Solução:**
1. **NÃO fazer swap se STAGING não estiver OK**
2. Investigar problema em STAGING
3. Corrigir e fazer novo deploy para STAGING
4. Só fazer swap quando STAGING estiver 100% OK

---

### Problema: Error rate alto após deploy

**Sintoma:** Application Insights mostra error rate > 5%

**Ação:**
```bash
# 1. ROLLBACK IMEDIATO
az webapp deployment slot swap \
  --name appview-prod \
  --resource-group appview-rg \
  --slot production \
  --target-slot staging

# 2. Investigar causa
az webapp log tail --name appview-prod --resource-group appview-rg

# 3. Verificar métricas Application Insights
# (via portal Azure)

# 4. Comunicar teams
# (canal #prod-deploys)
```

---

### Problema: Integração externa quebrada

**Sintoma:** Alfresco/Jarvis/Pega não respondem

**Checklist:**
```bash
# 1. Verificar se é problema nosso ou deles
curl https://alfresco-api.company.com/health

# 2. Verificar credentials
az webapp config appsettings list --name appview-prod --resource-group appview-rg | grep ALFRESCO

# 3. Verificar logs de integração
az webapp log tail --name appview-prod --resource-group appview-rg | grep "AlfrescoClient"

# 4. Se problema deles: contatar time externo
# 5. Se problema nosso: investigar código/config
```

---

## Contatos Emergenciais

### Times

| Time | Responsável | Telefone | Email |
|------|-------------|----------|-------|
| **DevOps** | João Silva | +55 11 9xxxx-xxxx | joao@company.com |
| **Tech Lead** | Maria Santos | +55 11 9xxxx-xxxx | maria@company.com |
| **Operações** | Pedro Costa | +55 11 9xxxx-xxxx | pedro@company.com |
| **Suporte** | Ana Lima | +55 11 9xxxx-xxxx | ana@company.com |

### Sistemas Externos

| Sistema | Contato | Telefone | Email |
|---------|---------|----------|-------|
| **Alfresco** | Suporte Vendor | 0800-xxx-xxxx | suporte@alfresco.com |
| **Jarvis** | Equipe Cloud | interno | jarvis-team@company.com |
| **Pega** | Suporte 24/7 | 0800-xxx-xxxx | support@pega.com |

---

**Fim do Documento**
