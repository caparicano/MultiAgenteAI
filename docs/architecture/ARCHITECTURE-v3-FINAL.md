# MultiAgent AI — Arquitectura Final (v3)

> GitHub-first. Supabase apenas para execute-migration em projectos externos futuros.
> Plataforma **multi-projecto**, **multi-sessão**, **€0/mês**.
> **Status: PROPOSTA FINAL**

---

## Princípios desta versão

| Princípio | Decisão |
|-----------|---------|
| **GitHub-first** | Usar GitHub para tudo o que GitHub já faz bem |
| **Zero infra custom** | Sem Supabase Control Plane, sem Edge Functions de orquestração |
| **Zero custo** | Repo público → GitHub Actions ilimitadas → €0/mês |
| **Supabase interno em Lovable** | Nunca assumir acesso directo ao Supabase de projectos Lovable |
| **Supabase externo (futuro)** | Único caso onde Supabase pessoal (wqidcmapowfudwsazblj) é usado |

---

## Porquê mudar de Supabase Control Plane para GitHub

| Problema da arquitectura anterior | Solução GitHub-first |
|-----------------------------------|---------------------|
| Supabase free tier pausa após 7 dias sem actividade | GitHub nunca pausa |
| Dashboard custom (Lovable) para construir | GitHub Projects já existe |
| Edge Functions de orquestração para construir | GitHub Actions já existe |
| Auth (magic link) para configurar | GitHub login já está feito |
| Audit log append-only via triggers | GitHub Issue timeline é nativo e imutável |
| `updated_by` em falta → N1 | GitHub regista quem mudou cada label/comentário |
| Retry policy para definir → N2 | GitHub Actions tem retry nativo |
| Fluxo `cancelled` sem UI → N3 | Fechar Issue = cancelar (UI nativa) |
| Race condition em verdicts → N4 | Labels são idempotentes (não se duplicam) |

---

## Arquitectura

```
 ┌────────────────────────────────────────────────────────────────┐
 │                    LUIS (gate final)                            │
 │   GitHub.com (desktop/web) │ Telegram (notificações + approve) │
 └──────────┬─────────────────┴──────────────────────────────────┘
            │
 ┌──────────▼──────────────────────────────────────────────────────┐
 │                    GITHUB (backbone único)                       │
 │                                                                  │
 │  Issues          → Tasks (corpo estruturado + labels)           │
 │  Labels          → Estado, prioridade, projecto, classificação  │
 │  Issue comments  → Planos, verdicts, AI reviews, SQL drafts     │
 │  Issue timeline  → Audit log (append-only, imutável por design) │
 │  Projects board  → Dashboard (filtros por projecto, estado)     │
 │  Actions         → Toda a automação (cron, API calls, CI)       │
 │  Secrets         → PERPLEXITY_API_KEY, TELEGRAM_BOT_TOKEN, etc  │
 │                                                                  │
 │  Workflows:                                                      │
 │    orchestrate.yml          (state machine via label events)    │
 │    review-perplexity.yml    (AI review → comment + label)       │
 │    notify-telegram.yml      (Telegram push nos gates)           │
 │    gate-timeout.yml         (cron 15min → auto-block expirados) │
 │    ci.yml                   (typecheck, tests — já existe)      │
 │    chatmd-conflict.yml      (detecta conflitos no chat.md)      │
 │    dispatch-prepare.yml     (prepara dispatch — Fase 3)         │
 └───┬──────────────────────┬──────────────┬────────────────────────┘
     │                      │              │
     ▼                      ▼              ▼
 ┌────────┐          ┌──────────┐   ┌──────────────────┐
 │ Claude │          │ Lovable  │   │  Perplexity API  │
 │ Code   │          │ (lê/     │   │  (sonar-reason.  │
 │        │          │  escreve  │   │   -pro)          │
 │ cria   │          │  chat.md  │   │                  │
 │ issues │          │  + push)  │   │  chamado via     │
 │ comenta│          │          │   │  GitHub Action   │
 │ labela │          │          │   │                  │
 └──┬─────┘          └─────┬────┘   └──────────────────┘
    │                      │
    │  ┌───────────────────┴──────────────────────────────────┐
    │  │         AGENT DISPATCH (Fase 3 — adicional)          │
    │  │                                                       │
    │  │  Camada OPCIONAL que delega interacções manuais       │
    │  │  a motores de execução AI.                            │
    │  │                                                       │
    │  │  Motor 1: Claude Browser Control                      │
    │  │    → Abre Lovable no browser do Luis                  │
    │  │    → Escreve no "Ask Lovable..." chat                 │
    │  │    → Cola SQL no Lovable SQL Editor                   │
    │  │    → Lê respostas, verifica builds                    │
    │  │                                                       │
    │  │  Motor 2: Perplexity API (análise)                    │
    │  │    → Pre-flight: valida instrução antes de enviar     │
    │  │    → Post-flight: valida resultado após execução      │
    │  │    → NÃO interage com Lovable UI                      │
    │  │                                                       │
    │  │  Trigger: /dispatch-* no Issue (Luis acciona)         │
    │  │  Fallback: se dispatch falha → fluxo manual base      │
    │  └───────────────────┬──────────────────────────────────┘
    │                      │
    │         ┌────────────▼─────────────┐
    │         │  SUPABASE INTERNO        │
    │         │  (Lovable-managed)       │
    │         │                          │
    │         │  get2event, capi, hr     │
    │         │                          │
    │         │  ← Lovable escreve       │
    │         │  ← Luis cola SQL (base)  │
    │         │  ← OU dispatch cola SQL  │
    │         │    (Fase 3, opcional)     │
    │         └──────────────────────────┘
    │
    │         ┌──────────────────────────────────────┐
    │         │  SUPABASE PESSOAL                    │
    │         │  (wqidcmapowfudwsazblj)              │
    │         │                                      │
    │         │  Usado APENAS para:                  │
    │         │  execute-migration (Fase 2)          │
    │         │  em projectos com Supabase EXTERNO   │
    │         │  (projectos futuros)                 │
    │         │                                      │
    │         │  NÃO usado para orquestração         │
    │         └──────────────────────────────────────┘
```

---

## Papéis

| Componente | Função | Dispatch (Fase 3) |
|------------|--------|--------------------|
| **Luis** | Gate final — aprova via GitHub Issues ou Telegram | Acciona dispatch via `/dispatch-*` |
| **Claude Code** | Cria Issues, comenta planos, adiciona labels, commita ao repo | **Motor de dispatch**: controla browser → interage com Lovable chat + SQL Editor |
| **Perplexity** | Chamado por GitHub Action → posta review como comment no Issue | **Análise pre/post-flight**: valida instruções e resultados antes/depois do dispatch |
| **Lovable** | Lê `lovable/chat.md`, implementa, faz push de commits | Recebe instruções via dispatch (Claude browser) em vez de manualmente |
| **GitHub Actions** | Orquestra tudo: transitions, AI calls, Telegram, timeouts, CI | `dispatch-prepare.yml` prepara e valida pedidos de dispatch |
| **GitHub Projects** | Dashboard visual (filtros nativos, board por estado) | — |
| **Supabase pessoal** | Só para execute-migration em projectos externos (Fase 2, futura) | — |

---

## State Machine via Labels

```
status:intake
    │ Action: orchestrate.yml detecta novo Issue
    ▼
status:planning
    │ Claude Code gera plano → comenta Issue
    │ Action: review-perplexity.yml chama API → comenta score + findings
    │ Action: notify-telegram.yml → push "Gate 1 pendente [projeto] [priority]"
    ▼
status:plan-review  ◄── GATE 1 (Luis)
    │ Luis comenta /approve ou /changes ou fecha Issue (cancel)
    ▼
status:execution
    │ Claude Code commita código, escreve INBOX em lovable/chat.md
    │ Luis diz ao Lovable "executa" (bridge manual)
    │ Lovable implementa, faz push
    │ GitHub CI corre (tsc, tests)
    │ Action: verifica scope drift (files_planned vs files_actual no PR)
    │ Action: notify-telegram.yml → push "Gate 2 pendente"
    ▼
status:code-review  ◄── GATE 2 (Luis)
    │ Luis comenta /approve ou /changes
    ▼
status:migration  (só se task tem SQL)
    │ Claude Code posta SQL no comment do Issue
    │ Luis copia para Lovable SQL Editor (bridge manual permanente)
    │ Luis comenta /migration-done
    ▼
status:verify  ◄── GATE 3 (Luis)
    │ Luis comenta /verify-ok
    ▼
status:done
    │ Issue fechado automaticamente

status:blocked  ← auto-block por timeout ou scope drift
    │ Luis pode reabrir: remove label + adiciona label anterior
    │ Ou fechar (cancel)
```

### Comandos de slash no Issue (Luis digita em comment)

**Base (Fases 1-2):**
| Comando | Efeito |
|---------|--------|
| `/approve` | Aprova gate actual → transita para próximo estado |
| `/changes [texto]` | Pede alterações → volta ao estado anterior |
| `/migration-done` | Confirma SQL aplicado → vai para verify |
| `/verify-ok` | Confirma verificação → fecha Issue como done |
| `/block [motivo]` | Bloqueia manualmente |
| `/cancel` | Fecha Issue como cancelled |

**Dispatch (Fase 3):**
| Comando | Efeito | Motor |
|---------|--------|-------|
| `/dispatch-lovable` | Claude abre Lovable chat, envia INBOX, lê resposta | Claude Browser |
| `/dispatch-sql` | Claude abre Lovable SQL Editor, cola SQL, executa | Claude Browser |
| `/dispatch-review` | Perplexity analisa instrução antes de dispatch | Perplexity API |
| `/dispatch-review-sql` | Perplexity analisa SQL antes de dispatch | Perplexity API |
| `/dispatch-lovable --with-review` | Pre-flight + dispatch + post-flight combinados | Ambos |
| `/dispatch-sql --with-review` | Perplexity valida SQL + Claude aplica + Perplexity verifica | Ambos |
| `/dispatch-research [query]` | Perplexity faz research contextualizado à task | Perplexity API |
| `/dispatch-abort` | Cancela dispatch em progresso | — |
| `/dispatch-skip` | Marca task como "dispatch skipped" → fluxo manual base | — |

---

## Estrutura de um Issue (Task)

```markdown
---
project: get2event
classification: complex
priority: high
assigned_agents: [claude_code, lovable]
files_affected_planned: [src/foo.ts, src/bar.ts]
---

## Descrição
[descrição da task]

---
<!-- Secções abaixo preenchidas pelos agentes durante o workflow -->

## Plano
[gerado por Claude Code]

## SQL Draft
[gerado por Claude Code, se aplicável]

## Lovable INBOX
[gerado por Claude Code — mensagem para chat.md]
```

**Labels num Issue típico:**
```
project:get2event  classification:complex  priority:high
status:execution   agent:claude_code       agent:lovable
```

---

## Labels do Sistema

### Status (estado da task)
```
status:intake          status:planning        status:plan-review
status:execution       status:code-review     status:migration
status:verify          status:done            status:blocked
status:cancelled
```

### Projecto
```
project:get2event    project:capi    project:hr    project:[novo]
```

### Classificação
```
classification:routine    classification:complex    classification:critical
```

### Prioridade
```
priority:low    priority:medium    priority:high    priority:critical
```

### Agentes
```
agent:claude_code    agent:lovable    agent:perplexity
```

### Gates e reviews
```
gate1:approved    gate1:changes    gate2:approved    gate2:changes
gate3:approved
perplexity:score-1  ...  perplexity:score-10
scope:clean    scope:drift-detected
timeout:gate1    timeout:gate2    timeout:gate3
```

### Dispatch (Fase 3)
```
dispatch:ready           dispatch:in-progress      dispatch:success
dispatch:failed          dispatch:needs-review     dispatch:skipped
dispatch:preflight-ok    dispatch:preflight-fail
```

---

## GitHub Actions (6 workflows base + 1 dispatch)

### `orchestrate.yml`
**Trigger:** `issues` events (labeled, unlabeled, commented)
**Lógica:**
- Detecta label adicionado → valida transição válida → adiciona novo label de status
- Detecta slash commands (`/approve`, `/changes`, etc.) → executa transição
- Escreve entry de auditoria como comment colapsável no Issue
- Em falha: adiciona label `status:blocked` com reason

### `review-perplexity.yml`
**Trigger:** label `status:planning` adicionado
**Lógica:**
1. Lê corpo do Issue + comentários de plano
2. POST `api.perplexity.ai/chat/completions` (model: `sonar-reasoning-pro`)
3. Posta resultado como comment estruturado:
   ```
   ## 🤖 Perplexity Review — Score: 8/10
   **Aprovado:** ✅
   **Findings:**
   - ...
   ```
4. Adiciona label `perplexity:score-8`
5. Transita → `status:plan-review`
6. Trigger `notify-telegram.yml`

### `notify-telegram.yml`
**Trigger:** workflow_call (chamado por outros workflows nos gates)
**Lógica:**
- Envia mensagem com: projecto, título, prioridade, estado actual, link para Issue
- Inclui contagem de gates pendentes: "3 tasks aguardam aprovação"
- Retry: 3 tentativas com backoff exponencial (1s, 4s, 16s)
- Falha total → label `notification:failed` no Issue (não bloqueia workflow)

### `gate-timeout.yml`
**Trigger:** `schedule: cron: '*/15 * * * *'`
**Lógica:**
- Busca Issues com labels `status:plan-review`, `status:code-review`, `status:verify`
- Verifica `updated_at` + timeout da classificação:
  - routine: 4h | complex: 24h | critical: 72h
- Se expirado: adiciona `status:blocked` + `timeout:gate{N}` + comenta motivo + notifica Telegram

### `chatmd-conflict.yml`
**Trigger:** push que altera `lovable/chat.md`
**Lógica:**
- Lê STATUS header do ficheiro
- Verifica quem fez o commit (Claude Code vs Lovable)
- Se conflito detectado → notifica Telegram + comenta no Issue activo

### `ci.yml` (já existe nos repos alvo)
**Trigger:** push, pull_request
**Lógica existente:** typecheck, tests
**Adição:** scope drift check — compara `files_affected_planned` (do Issue body) com `git diff --name-only` do PR. Se drift: adiciona label `scope:drift-detected` + comenta diff no Issue.

---

## Protocolo de Comunicação

### Com Lovable (ficheiros — único canal possível)

**chat.md com protocolo de lock:**
```markdown
## STATUS: CLEAR | INBOX_PENDING | OUTBOX_READY

## Issue
#42 — [título da task]

## INBOX (para o Lovable executar)
[mensagem gerada por Claude Code]

## OUTBOX (resposta do Lovable)
[preenchido pelo Lovable após execução]
```

`.orchestra/state.json` (legível por todos os agents):
```json
{
  "active_issue": 42,
  "project": "get2event",
  "status": "execution",
  "branch": "orchestra/issue-42",
  "last_updated": "2026-05-07T10:00:00Z"
}
```

### Com Claude Code
- `gh issue create` — cria tasks
- `gh issue comment` — adiciona planos, SQL, INBOX drafts
- `gh issue edit --add-label` — transita estado
- `gh issue list --label project:get2event` — lista tasks activas
- Tudo via `gh` CLI já configurado

### Com Perplexity
- Chamado exclusivamente por `review-perplexity.yml` via GitHub Actions Secret `PERPLEXITY_API_KEY`
- Nunca chamado directamente pelo Claude Code (evita inconsistência de onde o resultado é guardado)

---

## Dashboard: GitHub Projects

Configuração do board para o projecto `MultiAgenteAI`:

**Campos custom:**
- `Project` (text) — get2event / capi / hr / ...
- `Classification` (single select) — routine / complex / critical
- `Priority` (single select) — low / medium / high / critical
- `Perplexity Score` (number) — 1-10

**Views:**
1. **Board** — colunas = states (intake → done), cards = Issues, filtros por projecto
2. **Table** — todas as tasks com campos, sorted por priority desc + updated_at asc
3. **Timeline** — por data de criação

**Auth:** Luis já está logado no GitHub — zero setup adicional.

---

## Gestão de Secrets

Todos os secrets em **GitHub Actions Secrets** do repo `MultiAgenteAI`:

| Secret | Usado por | Rotação |
|--------|-----------|---------|
| `PERPLEXITY_API_KEY` | `review-perplexity.yml` | Manual quando necessário |
| `TELEGRAM_BOT_TOKEN` | `notify-telegram.yml` | Manual quando necessário |
| `TELEGRAM_CHAT_ID` | `notify-telegram.yml` | Estático |
| `GH_PAT` | Actions que criam Issues/comments cross-repo | Renovar anualmente |

Para `execute-migration` (Fase 2, projectos externos futuros):
- `SUPABASE_URL_{PROJECT}` e `SUPABASE_SERVICE_ROLE_{PROJECT}` nos secrets
- Usados apenas no workflow `execute-migration.yml` (Fase 2)

---

## Defesas antes de aplicar SQL (Supabase interno — manual)

1. Issue tem `gate2:approved` no seu histórico de labels
2. Perplexity review comment existe no Issue
3. Claude Code postou SQL num comment com bloco ` ```sql ` marcado
4. Luis copia SQL → Lovable SQL Editor → executa
5. Luis comenta `/migration-done` no Issue → workflow regista
6. CI verificação final (schema drift check)

Para Supabase **externo** (Fase 2):
- `execute-migration.yml` chamado via workflow_dispatch com `issue_number`
- Usa `SUPABASE_SERVICE_ROLE_{PROJECT}` secret
- Pre/post gate queries em transacção
- Resultado postado como comment no Issue

---

## Multi-projecto

Cada projecto é uma label `project:X`. Um Issue pertence a exactamente um projecto.

**Setup de novo projecto:**
1. Criar label `project:[nome]` no repo
2. Criar arquivo `.orchestra/projects/[nome].json` com config:
   ```json
   {
     "github_repo": "tech2c-devteam/[nome]",
     "supabase_mode": "internal",
     "lovable_project_id": "uuid-do-projecto-lovable",
     "lovable_project_url": "https://lovable.dev/projects/uuid-do-projecto-lovable",
     "lovable_chat_path": "lovable/chat.md",
     "orchestra_path": ".orchestra",
     "dispatch_enabled": false
   }
   ```
3. Adicionar `orchestra.yml` ao repo do projecto (webhook → `notify-telegram.yml`)

> **Nota Fase 3:** O campo `lovable_project_id` e `lovable_project_url` são necessários para o Agent Dispatch. O campo `dispatch_enabled` activa/desactiva o dispatch por projecto (default: `false`).

**Configuração existente de projectos (Supabase interno):**
```
.orchestra/projects/get2event.json  → supabase_mode: "internal"
.orchestra/projects/capi.json       → supabase_mode: "internal"
.orchestra/projects/hr.json         → supabase_mode: "internal"
```

---

## Automação vs Manual

| Step | Base (Fases 1-2) | Com Dispatch (Fase 3) |
|------|-------------------|------------------------|
| Criar task | Manual: Luis via `gh issue create` | Igual |
| Planning (plano + SQL) | Manual: Claude Code comenta no Issue | Igual |
| Perplexity AI review | **Auto**: `review-perplexity.yml` | Igual |
| **Gate 1 (plan review)** | **MANUAL**: Luis `/approve` ou `/changes` | Igual (gate humano, by design) |
| Gerar INBOX para Lovable | Manual: Claude Code comenta + commita `chat.md` | Igual |
| **Trigger Lovable** | **MANUAL**: Luis diz "executa" no Lovable | **DISPATCH**: `/dispatch-lovable` → Claude browser |
| CI (tsc, tests) | **Auto**: `ci.yml` | Igual |
| Scope drift check | **Auto**: `ci.yml` extensão | Igual + dispatch verifica pós-execução |
| **Gate 2 (code review)** | **MANUAL**: Luis `/approve` ou `/changes` | Igual (gate humano, by design) |
| Aplicar SQL (Supabase interno) | **MANUAL**: Luis copia SQL → SQL Editor | **DISPATCH**: `/dispatch-sql` → Claude browser |
| Aplicar SQL (Supabase externo) | **Auto (Fase 2)**: `execute-migration.yml` | Igual |
| **Gate 3 (verify)** | **MANUAL**: Luis `/verify-ok` | Igual (gate humano, by design) |
| Gate timeouts | **Auto**: `gate-timeout.yml` | Igual |
| Notificações Telegram | **Auto**: `notify-telegram.yml` | Igual + notificações de dispatch |
| Cancelar task | Manual: Luis fecha o Issue | Igual |
| Audit log | **Auto**: GitHub Issue timeline | Igual + dispatch reports como comments |
| **Pre-flight review** | — (não existe) | **Fase 3B**: Perplexity valida instrução antes de dispatch |
| **Post-flight review** | — (não existe) | **Fase 3B**: Perplexity valida resultado após dispatch |

### Bridges irredutíveis (qualquer fase)
1. **Trigger Lovable** — Lovable não tem API *(mitigável com Agent Dispatch — Fase 3)*
2. **Aplicar SQL em Supabase interno** — sem acesso externo ao Supabase do Lovable *(mitigável com Agent Dispatch — Fase 3)*
3. **Gates humanos** — by design, Luis é o gate final *(irredutível permanentemente)*

---

## Feature Adicional: Agent Dispatch (Fase 3)

> Camada **opcional** sobre o fluxo base. Permite delegar interacções manuais (trigger
> Lovable, aplicar SQL no SQL Editor) a motores de execução AI via browser control.
>
> **O fluxo base funciona SEM esta feature.** O dispatch é opt-in, activado por Luis
> task a task. Se o dispatch falha, volta ao fluxo manual base sem perda de estado.

---

### Porquê esta feature é necessária

O fluxo base tem 3 bridges manuais. Dois deles exigem que Luis abra o Lovable e faça
operações repetitivas:

| Bridge manual | Tempo por ocorrência | Frequência |
|---------------|---------------------|------------|
| Trigger Lovable (copiar INBOX, colar no chat, enviar) | ~3-5 min | Cada task com Lovable |
| Aplicar SQL (copiar SQL, abrir SQL Editor, colar, executar, verificar) | ~5-10 min | Cada task com SQL |
| Gates humanos (ler, decidir, comentar) | ~2-5 min | 1-3x por task |

O Agent Dispatch automatiza os dois primeiros. O terceiro mantém-se manual por design.

```
Fluxo BASE:
  Task ready → Telegram notifica → Luis abre Lovable → executa manualmente → confirma

Fluxo com DISPATCH:
  Task ready → Telegram notifica → Luis comenta /dispatch-lovable no Issue
  → Claude abre Lovable no browser → executa → verifica → reporta no Issue
  → Luis confirma resultado (gate humano mantém-se)
```

---

### Princípios de Design do Dispatch

| Princípio | Detalhe |
|-----------|---------|
| **Adicional** | O fluxo base funciona sem dispatch — é uma camada extra |
| **Opt-in por task** | Activado via `/dispatch-*` — não é global nem automático |
| **Opt-in por projecto** | `dispatch_enabled: true` no `.orchestra/projects/{project}.json` |
| **Human-gated** | Luis acciona o dispatch manualmente — não corre sozinho |
| **Verificado** | Resultado do dispatch é sempre verificado antes de prosseguir |
| **Fallback** | Se dispatch falha → volta ao fluxo manual base, sem perda de estado |
| **Transparente** | Todas as acções do dispatch são logadas como comments no Issue |
| **Não-destrutivo** | Dispatch nunca faz DELETE, DROP, ou ALTER destrutivo sem gate humano |

---

### Arquitectura do Dispatch

```
┌──────────────────────────────────────────────────┐
│            GitHub Issue #42                       │
│  status:execution                                │
│                                                  │
│  Luis comenta: /dispatch-lovable                 │
│  (ou /dispatch-sql, /dispatch-review)            │
└────────────────┬─────────────────────────────────┘
                 │
      ┌──────────▼──────────┐
      │  dispatch-prepare   │ ← GitHub Action
      │  .yml               │
      │                     │
      │  1. Valida estado   │
      │  2. Extrai instrução│
      │  3. Valida pré-cond.│
      │  4. Posta dispatch  │
      │     request comment │
      │  5. Label:          │
      │     dispatch:ready  │
      └──────────┬──────────┘
                 │
    ┌────────────▼────────────────────────┐
    │  Claude Code (sessão local do Luis) │
    │                                      │
    │  Detecta dispatch:ready no Issue     │
    │  (via polling ou notificação)        │
    └──────┬───────────────┬──────────────┘
           │               │
    ┌──────▼──────┐  ┌─────▼──────────────┐
    │   Motor 1   │  │     Motor 2        │
    │   CLAUDE    │  │     PERPLEXITY     │
    │   BROWSER   │  │     API            │
    │             │  │                    │
    │ Controla    │  │ Pre-flight:        │
    │ browser     │  │  valida instrução  │
    │ do Luis     │  │                    │
    │             │  │ Post-flight:       │
    │ Interage    │  │  valida resultado  │
    │ com Lovable │  │                    │
    │ chat + SQL  │  │ NÃO interage      │
    │ Editor      │  │ com browser        │
    └──────┬──────┘  └─────┬──────────────┘
           │               │
    ┌──────▼───────────────▼──────────────┐
    │  GitHub Issue #42                    │
    │                                      │
    │  Comment: resultado do dispatch      │
    │  Label: dispatch:success/failed/     │
    │         needs-review                 │
    │  Telegram: notificação do resultado  │
    └──────────────────────────────────────┘
```

---

### Motor 1: Claude Browser Control

#### Pré-requisitos

| Requisito | Detalhe | Verificação |
|-----------|---------|-------------|
| Claude Code activo | Sessão local com browser MCP (Claude in Chrome) | Claude está a correr neste terminal |
| Browser autenticado no Lovable | Luis fez login no Lovable no browser | Screenshot da página Lovable mostra dashboard |
| `lovable_project_id` configurado | `.orchestra/projects/{project}.json` tem o ID | Ficheiro existe e tem URL válido |
| Issue com instrução | Secção "Lovable INBOX" ou bloco `sql` no Issue | Issue body/comments têm conteúdo |
| Gate anterior aprovado | `gate1:approved` (ou gate relevante) presente | Label existe no Issue |
| Dispatch enabled | `dispatch_enabled: true` no config do projecto | Campo no JSON é `true` |

#### Protocolo A: Interacção com Lovable Chat ("Ask Lovable...")

Sequência de 7 fases, cada uma com critérios de sucesso e modos de falha:

```
FASE 1 — PREPARAÇÃO (local, sem browser)
├── Ler Issue body → extrair secção "Lovable INBOX"
├── Ler .orchestra/projects/{project}.json → obter lovable_project_url
├── Validar instrução não vazia e < 10.000 caracteres
├── Validar que Issue tem gate anterior aprovado
│
├── ✅ Sucesso: instrução válida + URL válido + gates OK
├── ❌ Falha: campo vazio → ABORT + comment "Dispatch abortado: INBOX vazio"
└── ❌ Falha: gate em falta → ABORT + comment "Dispatch abortado: gate não aprovado"

FASE 2 — NAVEGAÇÃO
├── Abrir tab para lovable_project_url
├── Aguardar page load (timeout: 30s)
├── Screenshot → verificar visualmente que:
│   ├── Estamos no projecto correcto (nome visível)
│   ├── Chat input "Ask Lovable..." está visível
│   └── Não há modal de login/captcha
│
├── ✅ Sucesso: página carregada, chat visível
├── ❌ Falha: login required → ABORT + "Sessão Lovable expirada — requer login manual"
├── ❌ Falha: timeout → RETRY 1x → ABORT + "Lovable não carregou"
└── ❌ Falha: projecto errado → ABORT + "URL aponta para projecto errado"

FASE 3 — ENVIO DA INSTRUÇÃO
├── Encontrar input "Ask Lovable..." (via find ou read_page)
├── Clicar no input para focus
├── Escrever instrução completa
│   ├── Se instrução < 2000 chars: colar de uma vez
│   └── Se instrução > 2000 chars: dividir em chunks de 1500 chars
├── Screenshot → verificar que instrução está no input
├── Clicar Send (Enter ou botão)
│
├── ✅ Sucesso: instrução enviada, Lovable começou a processar
├── ❌ Falha: input não encontrado → Screenshot + ABORT "UI Lovable mudou"
└── ❌ Falha: instrução truncada → Screenshot + ABORT "Instrução não coube"

FASE 4 — AGUARDAR RESPOSTA DO LOVABLE
├── Polling a cada 15 segundos:
│   ├── Screenshot → verificar se resposta apareceu
│   ├── Detectar indicadores de "processando" (loading spinner, typing)
│   └── Detectar indicadores de conclusão (resposta completa visível)
├── Timeout configurável (default: 5 minutos, max: 15 minutos)
│
├── ✅ Sucesso: Lovable respondeu com texto legível
├── ❌ Falha: timeout → ABORT + "Lovable não respondeu em {N} minutos"
└── ⚠️ Parcial: Lovable respondeu com "I'll work on this" → continuar polling

FASE 5 — LER E CLASSIFICAR RESULTADO
├── Extrair texto completo da resposta do Lovable
├── Screenshot da resposta (para audit)
├── Classificar resposta:
│   ├── SUCCESS: Lovable diz que completou + sem erros visíveis
│   ├── PARTIAL: Lovable completou parte, pede input adicional
│   ├── BUILD_FAILED: Lovable reporta erros de build
│   ├── LOVABLE_ERROR: Lovable diz que não consegue fazer
│   └── AMBIGUOUS: resposta não é clara sobre sucesso/falha
│
├── ✅ SUCCESS → prosseguir para verificação
├── ⚠️ PARTIAL/AMBIGUOUS → label dispatch:needs-review + Telegram
├── ❌ BUILD_FAILED → label dispatch:failed + detalhes no comment
└── ❌ LOVABLE_ERROR → label dispatch:failed + resposta integral no comment

FASE 6 — VERIFICAÇÃO PÓS-EXECUÇÃO
├── Aguardar 30-60s para Lovable fazer push ao GitHub
├── Verificar git: `gh pr list` ou `git log` no repo do projecto
├── Se Lovable fez push:
│   ├── Ler diff: `git diff --name-only`
│   ├── Comparar com files_affected_planned do Issue
│   ├── Se match → scope OK
│   └── Se divergência → scope:drift-detected
├── Aplicar verificação OUTBOX (regra CLAUDE.md):
│   ├── Ler ficheiros reais (não confiar no que Lovable diz)
│   ├── Correr tsc --noEmit se possível
│   ├── Verificar casts suspeitos (as any, as unknown)
│   └── Comparar afirmações vs factos
│
├── ✅ Verificação OK → dispatch:success
├── ⚠️ Scope drift → dispatch:needs-review + scope:drift-detected
└── ❌ Build quebrado → dispatch:failed + detalhes

FASE 7 — REPORT
├── Postar comment estruturado no Issue:
│   │
│   │  ## 🤖 Dispatch Report — Claude Browser
│   │  **Status:** SUCCESS / FAILED / NEEDS_REVIEW
│   │  **Motor:** Claude Browser Control
│   │  **Protocolo:** Lovable Chat
│   │  **Duração:** {N} minutos
│   │  
│   │  ### Instrução enviada
│   │  <details><summary>Ver instrução</summary>
│   │  {instrução completa}
│   │  </details>
│   │  
│   │  ### Resposta do Lovable
│   │  <details><summary>Ver resposta</summary>
│   │  {resposta completa}
│   │  </details>
│   │  
│   │  ### Verificação
│   │  - Files pushed: {lista}
│   │  - Scope drift: ✅ clean / ⚠️ detected
│   │  - Build: ✅ pass / ❌ fail / ⏳ pending
│   │  
│   │  ### Screenshots
│   │  <details><summary>Ver screenshots</summary>
│   │  [screenshots das fases 2-5]
│   │  </details>
│   │
├── Actualizar label: dispatch:success / dispatch:failed / dispatch:needs-review
├── Se SUCCESS: transitar estado normalmente (remove status actual, adiciona próximo)
├── Se FAILED: adicionar status:blocked + notificar Telegram
└── Se NEEDS_REVIEW: manter estado actual + notificar Telegram "Review humano necessário"
```

#### Protocolo B: Interacção com Lovable SQL Editor

Usado quando a task envolve SQL migration em Supabase interno do Lovable.

```
FASE 1 — PREPARAÇÃO (local, sem browser)
├── Ler Issue comments → encontrar bloco ```sql``` mais recente
├── Validar SQL:
│   ├── Não está vazio
│   ├── Não contém DROP DATABASE, TRUNCATE sem WHERE, ou ALTER SYSTEM
│   ├── Contém BEGIN/COMMIT (transaccional)
│   └── Tamanho < 50KB
├── Ler gate queries (se existirem no comment seguinte ao SQL)
│
├── ✅ SQL válido + gate queries encontradas
├── ❌ SQL perigoso detectado → ABORT + "SQL contém operações destrutivas — requer manual"
└── ❌ SQL vazio → ABORT + "Nenhum bloco SQL encontrado no Issue"

FASE 2 — NAVEGAÇÃO AO SQL EDITOR
├── Navegar para lovable_project_url
├── Aguardar page load
├── Encontrar e clicar botão "Cloud" (ícone cloud no topo do editor)
├── Encontrar e clicar "SQL Editor" no menu Cloud
├── Aguardar SQL Editor carregar (textarea visível)
├── Screenshot → confirmar que estamos no SQL Editor
│
├── ✅ SQL Editor visível com textarea
├── ❌ Botão Cloud não encontrado → ABORT + "UI Lovable mudou — Cloud button missing"
├── ❌ SQL Editor não carregou → RETRY 1x → ABORT
└── ❌ Login required → ABORT + "Sessão expirada"

FASE 3 — COLAR E EXECUTAR SQL
├── Encontrar textarea do SQL Editor
├── Clicar para focus
├── Seleccionar tudo (Ctrl+A) → limpar conteúdo existente
├── Colar SQL completo
├── Screenshot → verificar que SQL foi colado correctamente
│   ├── Primeiras 3 linhas visíveis correspondem ao SQL?
│   └── Últimas linhas visíveis contêm COMMIT?
├── Encontrar e clicar botão "Run" / "Execute"
├── Aguardar resultado (timeout: 30s)
│
├── ✅ Resultado mostra "Success" ou "N rows affected"
├── ❌ Resultado mostra erro SQL → capturar erro + ABORT
└── ❌ Timeout → ABORT + "SQL Editor não respondeu"

FASE 4 — GATE QUERIES PÓS-EXECUÇÃO
├── Se existem gate queries:
│   ├── Limpar SQL Editor
│   ├── Colar gate query #1
│   ├── Executar → ler resultado
│   ├── Repetir para cada gate query
│   └── Comparar resultados com expectativas documentadas no Issue
├── Se não existem gate queries:
│   └── Reportar "Sem gate queries — verificação manual necessária"
│
├── ✅ Todas as gate queries passam → migration validada
├── ⚠️ Alguma gate query falha → dispatch:needs-review
└── ❌ Gate query mostra estado inesperado → dispatch:failed + ROLLBACK se possível

FASE 5 — VERIFICAÇÃO DE SCHEMA
├── Navegar para tab "Database" no Lovable Cloud
├── Screenshot das tabelas relevantes
├── Verificar que novas colunas/tabelas existem conforme esperado
│
├── ✅ Schema corresponde ao esperado
└── ⚠️ Divergência → dispatch:needs-review + detalhar divergência

FASE 6 — REPORT (formato igual ao Protocolo A, com secções adicionais)
│  ### SQL Executado
│  <details><summary>Ver SQL</summary>
│  {SQL completo}
│  </details>
│  
│  ### Gate Queries
│  | Query | Resultado esperado | Resultado real | Status |
│  |-------|-------------------|----------------|--------|
│  | SELECT count(*) FROM ... | 1 | 1 | ✅ |
│  
│  ### Schema verification
│  [screenshots da tab Database]
```

---

### Motor 2: Perplexity API (Análise Pre/Post-flight)

O Perplexity **NÃO controla browser**. O seu papel no dispatch é puramente analítico —
funciona como segundo revisor que valida o que vai ser enviado e o que foi recebido.

#### O que pode fazer

| Acção | Trigger | Input | Output |
|-------|---------|-------|--------|
| **Pre-flight review** | `/dispatch-review` antes do dispatch | Instrução INBOX + contexto da task | Score 1-10 + risks + sugestões |
| **SQL validation** | `/dispatch-review-sql` | Bloco SQL + schema context | Score 1-10 + SQL risks + side effects |
| **Post-flight review** | Automático após dispatch | Resposta do Lovable + diff | Score 1-10 + discrepâncias detectadas |
| **Research** | `/dispatch-research [query]` | Pergunta de implementação | Análise + citações + recomendações |

#### O que NÃO pode fazer

- ❌ Navegar para Lovable ou qualquer site autenticado
- ❌ Escrever no chat do Lovable
- ❌ Executar SQL
- ❌ Controlar browser
- ❌ Aceder a dados privados do projecto (só recebe o que lhe é enviado)

#### Prompt template para pre-flight

```
System: You are a code reviewer for a multi-agent AI orchestration system.
You are reviewing an instruction that will be sent to Lovable (an AI code 
assistant) to implement. Evaluate the instruction for:
1. Clarity — will Lovable understand exactly what to do?
2. Scope — is the instruction well-bounded or will it cause scope creep?
3. Safety — are there any destructive operations or risky patterns?
4. Completeness — does the instruction include all needed context?

Return JSON:
{
  "score": 1-10,
  "approved": true/false,
  "risks": ["risk1", ...],
  "suggestions": ["suggestion1", ...],
  "scope_assessment": "bounded|vague|risky"
}

User: 
Task: {issue_title}
Project: {project_name}
Classification: {classification}
Files planned: {files_affected_planned}
Instruction to Lovable:
---
{inbox_content}
---
```

#### Integração combinada: Perplexity + Claude dispatch

```
MODO CONSERVADOR (classification: critical)
├── 1. /dispatch-review → Perplexity pre-flight
├── 2. Se score < 7 → STOP, notificar Luis
├── 3. Se score ≥ 7 → /dispatch-lovable → Claude browser
├── 4. Lovable executa
├── 5. Post-flight automático → Perplexity analisa resultado
├── 6. Se score < 7 → dispatch:needs-review
└── 7. Se score ≥ 7 → dispatch:success

MODO STANDARD (classification: complex)
├── 1. /dispatch-lovable → Claude browser (sem pre-flight)
├── 2. Lovable executa
├── 3. Post-flight automático → Perplexity analisa resultado
└── 4. Resultado reportado no Issue

MODO RÁPIDO (classification: routine)
├── 1. /dispatch-lovable → Claude browser (sem pre/post-flight)
└── 2. Resultado reportado no Issue

COMBINADO (qualquer classificação)
├── 1. /dispatch-lovable --with-review
├── 2. Perplexity pre-flight → resultado no Issue
├── 3. Se pre-flight OK → Claude browser dispatch
├── 4. Perplexity post-flight → resultado no Issue
└── 5. Report consolidado
```

---

### Slash Commands de Dispatch

| Comando | Efeito | Motor usado |
|---------|--------|-------------|
| `/dispatch-lovable` | Claude abre Lovable chat, envia INBOX, lê resposta | Claude Browser |
| `/dispatch-sql` | Claude abre Lovable SQL Editor, cola SQL, executa | Claude Browser |
| `/dispatch-review` | Perplexity analisa instrução antes de dispatch | Perplexity API |
| `/dispatch-review-sql` | Perplexity analisa SQL antes de dispatch | Perplexity API |
| `/dispatch-lovable --with-review` | Perplexity pre-flight + Claude dispatch + Perplexity post-flight | Ambos |
| `/dispatch-sql --with-review` | Perplexity valida SQL + Claude aplica + Perplexity verifica | Ambos |
| `/dispatch-research [query]` | Perplexity faz research contextualizado à task | Perplexity API |
| `/dispatch-abort` | Cancela dispatch em progresso | — |
| `/dispatch-skip` | Marca task como "dispatch skipped" → fluxo manual base | — |

---

### Labels de Dispatch

```
dispatch:ready           → Dispatch preparado e validado (Action já correu)
dispatch:in-progress     → Dispatch em execução (Claude browser a trabalhar)
dispatch:success         → Dispatch completou com sucesso
dispatch:failed          → Dispatch falhou (ver comment para detalhes)
dispatch:needs-review    → Dispatch completou mas resultado ambíguo — Luis decide
dispatch:skipped         → Luis optou por fluxo manual base
dispatch:preflight-ok    → Perplexity pre-flight score ≥ 7
dispatch:preflight-fail  → Perplexity pre-flight score < 7 — requer decisão humana
```

---

### Workflow: `dispatch-prepare.yml` (Fase 3)

**Trigger:** Issue comment contendo `/dispatch-*`
**Responsabilidade:** Validar e preparar o dispatch (NÃO executa — a execução é local)

```yaml
# Pseudo-código do workflow
on:
  issue_comment:
    types: [created]

jobs:
  dispatch-prepare:
    if: startsWith(github.event.comment.body, '/dispatch-')
    runs-on: ubuntu-latest
    steps:
      # 1. Parse do comando
      - parse: /dispatch-lovable | /dispatch-sql | /dispatch-review | etc.
      
      # 2. Validações
      - verify: Issue tem status:execution OU status:migration
      - verify: dispatch_enabled == true no project config
      - verify: gate anterior aprovado
      - verify: não existe dispatch:in-progress (anti-duplo)
      
      # 3. Extrair conteúdo
      - if dispatch-lovable: extrair "Lovable INBOX" do Issue body
      - if dispatch-sql: extrair bloco ```sql``` dos comments
      - if dispatch-review: preparar payload para Perplexity API
      
      # 4. Se /dispatch-review: chamar Perplexity API
      - if dispatch-review:
          POST api.perplexity.ai → score + findings
          post comment com resultado
          add label dispatch:preflight-ok OU dispatch:preflight-fail
          if preflight-fail: STOP (Luis decide)
      
      # 5. Preparar dispatch request
      - post comment:
          "## 🚀 Dispatch Request\n
           **Tipo:** lovable-chat / sql-editor / review\n
           **Motor:** Claude Browser / Perplexity API\n
           **Instrução:**\n```\n{content}\n```\n
           **Pré-condições:** ✅ gate approved, ✅ dispatch enabled\n
           **Status:** Aguarda execução local pelo Claude Code"
      
      # 6. Sinalizar pronto
      - add label: dispatch:ready
      - notify telegram: "Dispatch #42 pronto — /dispatch-lovable pendente execução"
```

---

### Error Handling Completo

| Erro | Detecção | Recovery | Label |
|------|----------|---------|-------|
| Sessão Lovable expirada | Screenshot mostra login page | ABORT → Telegram "Requer re-login no Lovable" | `dispatch:failed` |
| Lovable down (500/timeout) | Page load falha 2x | ABORT → Telegram "Lovable indisponível" | `dispatch:failed` |
| Lovable resposta timeout (>5min) | Timer excedido sem resposta | ABORT → reportar no Issue | `dispatch:failed` |
| Lovable build failed | Resposta contém erro de build | Reportar erro completo no Issue | `dispatch:failed` |
| SQL error no SQL Editor | Editor mostra erro SQL | Capturar erro + reportar | `dispatch:failed` |
| Scope drift (files errados) | `git diff` ≠ `files_affected_planned` | Reportar diff + label scope:drift | `dispatch:needs-review` |
| UI Lovable mudou (selector falha) | Elemento não encontrado via find | Screenshot + ABORT "UI mudou" | `dispatch:failed` |
| Lovable renomeia schema (drift) | Gate queries pós-SQL falham | CRITICAL: bloquear + Telegram urgente | `dispatch:failed` |
| Instrução ambígua | Lovable pede clarificação | Reportar pergunta do Lovable | `dispatch:needs-review` |
| Dispatch duplicado | Label `dispatch:in-progress` já existe | Rejeitar `/dispatch-*` | — (comment de rejeição) |
| Claude Code não activo | `dispatch:ready` fica sem execução >30min | `gate-timeout.yml` detecta → Telegram | `dispatch:failed` |

**Regra de ouro:** Qualquer erro no dispatch resulta em **fallback silencioso para o fluxo manual base**. O estado do Issue nunca fica corrupto — o dispatch apenas adiciona labels e comments, nunca remove labels de estado.

---

### Segurança do Dispatch

| Aspecto | Medida |
|---------|--------|
| **Credenciais Lovable** | NÃO armazenadas — dispatch usa sessão browser existente do Luis |
| **Scope do dispatch** | Limitado aos 2 protocolos (chat + SQL Editor) — nunca faz nada fora destes |
| **SQL destrutivo** | Fase 1 do Protocolo B rejeita DROP DATABASE, TRUNCATE sem WHERE, ALTER SYSTEM |
| **Gate obrigatório** | Dispatch só executa se gate anterior está aprovado no Issue |
| **Anti-duplo** | Label `dispatch:in-progress` impede dispatch simultâneo na mesma task |
| **Kill switch** | `/dispatch-abort` cancela qualquer dispatch em progresso |
| **Audit completo** | Cada acção do dispatch gera comment colapsável no Issue com screenshots |
| **ToS Lovable** | Uso supervisionado pelo humano (Luis acciona, Claude executa) — não é bot autónomo 24/7 |
| **Sem credenciais em Secrets** | Dispatch local não requer LOVABLE_USERNAME/PASSWORD nos GitHub Secrets |

---

### Fases de Implementação do Dispatch

#### Fase 3A — Dispatch manual via Claude Code (~1 weekend)

| Capacidade | Descrição |
|-----------|-----------|
| `/dispatch-lovable` | Claude Code (já em sessão) lê Issue, abre Lovable, executa Protocolo A |
| `/dispatch-sql` | Claude Code lê Issue, abre SQL Editor, executa Protocolo B |
| `dispatch-prepare.yml` | GitHub Action que valida e prepara dispatch requests |
| Labels de dispatch | Todos os labels `dispatch:*` criados |
| Report comment | Comment estruturado com screenshots no Issue |

**Como funciona na Fase 3A:**
1. Luis comenta `/dispatch-lovable` no Issue
2. `dispatch-prepare.yml` valida e adiciona `dispatch:ready`
3. Telegram notifica: "Dispatch #42 pronto"
4. Luis, que já tem Claude Code aberto, diz: "executa dispatch do Issue #42"
5. Claude Code lê o dispatch request, executa o protocolo browser
6. Resultado postado no Issue
7. Luis valida e comenta `/approve` (gate humano)

#### Fase 3B — Dispatch com Perplexity pre/post-flight (~1 weekend)

| Capacidade | Descrição |
|-----------|-----------|
| `/dispatch-review` | Perplexity pre-flight via API (integrado no dispatch-prepare.yml) |
| `/dispatch-lovable --with-review` | Modo combinado: pre-flight + dispatch + post-flight |
| Post-flight automático | Após dispatch success, Perplexity analisa resultado automaticamente |
| Modos por classificação | Conservador (critical), standard (complex), rápido (routine) |

#### Fase 3C — Dispatch semi-automático (futuro)

| Capacidade | Descrição |
|-----------|-----------|
| Auto-dispatch com opt-out | Task chega a `status:execution` → countdown 30min → dispatch automático |
| Telegram inline buttons | "Dispatch agora / Pular / Cancelar" directamente no Telegram |
| Dashboard custom (se existir) | Botões visuais por task |
| Batch dispatch | Dispatch múltiplas tasks em sequência |

---

### Limitações Conhecidas do Dispatch

| Limitação | Impacto | Mitigação |
|-----------|---------|-----------|
| **Requer máquina do Luis activa** | Dispatch não corre às 3h da manhã | Fase 3C: auto-dispatch durante horário de trabalho |
| **Requer sessão Lovable autenticada** | Se session expirar: dispatch falha | Notificação clara + fallback manual |
| **Fragilidade de UI do Lovable** | Se Lovable mudar UI: selectors partem | Screenshots no report ajudam a diagnosticar; fix é actualizar selectors |
| **Velocidade ~1-5 min por dispatch** | Mais lento que API directa | Aceitável — substitui 3-10 min manuais |
| **Uma operação de cada vez** | Não faz batch de 10 tasks | Fase 3C: batch dispatch sequencial |
| **Lovable pode reinterpretar** | Mesmo com instrução precisa, Lovable pode fazer scope creep | Verificação pós-execução obrigatória (Fase 6 do protocolo) |
| **Claude API computer use (server-side) é caro** | $3-8/task | Fase 3A usa Claude Code local (€0); server-side só na Fase 3C |
| **Screenshots não ficam no GitHub** | GitHub Issues não suporta upload via API facilmente | Screenshots ficam como base64 em comments colapsáveis ou links externos |

---

## Componentes a Construir

### Fase 1A — Motor (~1 weekend)

| # | Componente | Tipo |
|---|-----------|------|
| 1 | Labels no repo | GitHub UI | Criar todos os labels (status:*, project:*, priority:*, etc.) |
| 2 | `.orchestra/projects/*.json` | Ficheiros | Config de get2event, capi, hr |
| 3 | `orchestrate.yml` | GitHub Action | State machine: label events + slash commands |
| 4 | `notify-telegram.yml` | GitHub Action | Telegram push + retry |
| 5 | `gate-timeout.yml` | GitHub Action | Cron 15min + auto-block |
| 6 | Telegram Bot | @BotFather | Criar bot, obter token, adicionar a GitHub Secrets |

**Critério de saída Fase 1A:** 1 Issue real (ex: hotfix get2event) percorre todos os estados via slash commands + notificações Telegram. Timeline do Issue serve de audit trail.

### Fase 1B — AI + CI (~1 weekend)

| # | Componente | Tipo |
|---|-----------|------|
| 7 | `review-perplexity.yml` | GitHub Action | AI review automático + comment no Issue |
| 8 | `ci.yml` extensão | GitHub Action | Scope drift check (files_planned vs PR diff) |
| 9 | `chatmd-conflict.yml` | GitHub Action | Lock protocol para chat.md |
| 10 | GitHub Projects setup | GitHub UI | Board com custom fields + views |

**Critério de saída Fase 1B:** Luis vê Perplexity score no Issue antes de aprovar Gate 1. Scope drift é detectado e reportado automaticamente. Dashboard no GitHub Projects.

### Fase 2 — SQL automático para externos (~1 weekend)

| # | Componente | Tipo |
|---|-----------|------|
| 11 | `execute-migration.yml` | GitHub Action | SQL gated (só `supabase_mode: external`) via Supabase pessoal |

### Fase 3A — Agent Dispatch: Claude Browser (~1 weekend)

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 12 | Labels `dispatch:*` | GitHub UI | 8 labels de dispatch (ready, in-progress, success, failed, etc.) |
| 13 | `dispatch-prepare.yml` | GitHub Action | Valida `/dispatch-*`, extrai instrução, prepara dispatch request |
| 14 | Protocolo A: Lovable Chat | Protocolo Claude Code | Sequência de 7 fases para interagir com "Ask Lovable..." via browser |
| 15 | Protocolo B: SQL Editor | Protocolo Claude Code | Sequência de 6 fases para colar/executar SQL no Lovable Cloud |
| 16 | Dispatch report template | Comment template | Comment estruturado com status, instrução, resposta, verificação, screenshots |
| 17 | `lovable_project_id` nos configs | Ficheiros | Adicionar ID e URL do projecto Lovable a `.orchestra/projects/*.json` |

**Critério de saída Fase 3A:** Claude Code consegue ler Issue #N, abrir Lovable, enviar instrução via chat, ler resposta, e postar report no Issue — tudo via `/dispatch-lovable`.

### Fase 3B — Agent Dispatch: Perplexity Pre/Post-flight (~1 weekend)

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 18 | Pre-flight review | Extensão `dispatch-prepare.yml` | Chama Perplexity API para validar instrução (score + risks) |
| 19 | Post-flight review | Claude Code protocol | Após dispatch success, chama Perplexity para validar resultado |
| 20 | Modos por classificação | Lógica | Conservador (critical), standard (complex), rápido (routine) |
| 21 | Flag `--with-review` | Slash command | Combina pre-flight + dispatch + post-flight num único comando |

**Critério de saída Fase 3B:** `/dispatch-lovable --with-review` executa pre-flight Perplexity (score no Issue), dispatch Claude browser, e post-flight Perplexity (validação no Issue).

### Fase 3C — Agent Dispatch: Semi-automático (futuro)

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 22 | Auto-dispatch com opt-out | GitHub Action + Telegram | Task em `status:execution` → countdown 30min → dispatch automático |
| 23 | Telegram inline dispatch | Telegram Bot | Botões "Dispatch / Skip / Cancel" directamente no Telegram |
| 24 | Batch dispatch | Protocolo Claude Code | Dispatch sequencial de múltiplas tasks |
| 25 | Dashboard custom (opcional) | React SPA | Botões visuais "Execute via Claude" / "Review via Perplexity" por task |

### Fase 4 — Polish

| # | Componente | Descrição |
|---|-----------|-----------|
| 26 | Issue templates | Templates no GitHub para criar tasks com estrutura correcta |
| 27 | `classify.yml` | Action que auto-classifica Issues novos via Claude API |
| 28 | `auto-inbox.yml` | Action que commita `lovable/chat.md` automaticamente |

---

## Verificação

### Fase 1A — Happy path
1. `gh issue create` com labels `project:get2event status:intake priority:high classification:complex`
2. Action `orchestrate.yml` detecta → transita para `status:planning`
3. Claude Code comenta plano + SQL no Issue
4. Telegram recebe notificação "Gate 1 pendente — get2event [HIGH]"
5. Luis comenta `/approve` → Issue transita `status:plan-review → status:execution`
6. Timeline do Issue mostra todas as transições com actor + timestamp

### Fase 1A — Caminhos de falha
7. Luis comenta `/changes "falta cobertura de tests"` → Issue volta a `status:planning`
8. Issue fica em `status:plan-review` por 4h → `gate-timeout.yml` adiciona `status:blocked` + `timeout:gate1` → Telegram notifica
9. Luis comenta `/cancel` → Issue fechado como cancelled

### Multi-projecto
10. Criar Issues em `project:get2event` e `project:capi` simultaneamente
11. GitHub Projects board mostra ambos nos estados correctos, filtráveis por projecto

### Fase 3A — Dispatch happy path
12. Issue #N em `status:execution` com INBOX preenchido
13. Luis comenta `/dispatch-lovable`
14. `dispatch-prepare.yml` valida → adiciona `dispatch:ready` → Telegram notifica
15. Claude Code lê dispatch request → abre Lovable → envia instrução → lê resposta
16. Dispatch report postado como comment no Issue com screenshots
17. Label `dispatch:success` adicionado → estado transita normalmente
18. Luis comenta `/approve` (gate humano mantém-se)

### Fase 3A — Dispatch failure paths
19. Sessão Lovable expirada → dispatch:failed + "Requer re-login manual"
20. Lovable timeout (>5min) → dispatch:failed + fallback manual
21. Scope drift detectado pós-dispatch → dispatch:needs-review + scope:drift-detected
22. Luis comenta `/dispatch-abort` durante execução → dispatch cancelado
23. Luis comenta `/dispatch-skip` → dispatch:skipped, fluxo manual base

### Fase 3B — Dispatch com Perplexity
24. `/dispatch-lovable --with-review` → pre-flight score 8/10 → dispatch → post-flight score 9/10 → success
25. `/dispatch-review` com score < 7 → dispatch:preflight-fail → Luis decide se avança ou altera instrução

---

## Custo

### Fases 1-2 (base)

| Serviço | Custo |
|---------|-------|
| GitHub (repo público) | **€0** |
| GitHub Actions (repo público) | **€0 ilimitado** |
| GitHub Issues + Projects | **€0** |
| GitHub Secrets | **€0** |
| Perplexity API (~20 reviews/dia) | **~€3-5/mês** |
| Telegram Bot | **€0** |
| Supabase pessoal (só Fase 2+) | **€0** (já existe, activity mantém activo) |
| **Total Fases 1-2** | **~€3-5/mês** |

### Fase 3 (dispatch — adicional)

| Serviço | Custo | Notas |
|---------|-------|-------|
| Fase 3A: Claude Code local (browser dispatch) | **€0** | Usa sessão Claude Code existente do Luis |
| Fase 3B: Perplexity pre/post-flight | **~€1-2/mês extra** | Mais chamadas API, mesmo key |
| Fase 3C: Claude API computer use (server-side) | **~€3-8/task** | Só se quiser dispatch autónomo (sem Claude Code local) |
| Fase 3C: Dashboard custom (se Lovable) | **€0** | Build via Lovable no Supabase pessoal |
| **Total com Fase 3A+3B** | **~€5-7/mês** |
| **Total com Fase 3C (server-side)** | **~€15-50/mês** (depende do volume) |

---

## Próximos Passos

1. ⏳ Luis aprova arquitectura v3
2. **Fase 1A:**
   - [ ] Criar todos os labels no repo MultiAgenteAI
   - [ ] Criar `.orchestra/projects/get2event.json` + capi + hr
   - [ ] Criar Telegram Bot via @BotFather
   - [ ] Adicionar `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` + `PERPLEXITY_API_KEY` a GitHub Secrets
   - [ ] `orchestrate.yml` — state machine
   - [ ] `notify-telegram.yml` — notificações
   - [ ] `gate-timeout.yml` — cron
   - [ ] Teste end-to-end com 1 Issue real
3. **Fase 1B:**
   - [ ] `review-perplexity.yml` — AI review
   - [ ] Extensão `ci.yml` — scope drift
   - [ ] `chatmd-conflict.yml` — lock detection
   - [ ] GitHub Projects board setup
4. **Fase 2:**
   - [ ] `execute-migration.yml` (só quando existirem projectos externos)
5. **Fase 3A — Agent Dispatch: Claude Browser:**
   - [ ] Criar labels `dispatch:*` no repo
   - [ ] `dispatch-prepare.yml` — validação e preparação
   - [ ] Adicionar `lovable_project_id` aos configs `.orchestra/projects/*.json`
   - [ ] Implementar Protocolo A (Lovable Chat) no Claude Code
   - [ ] Implementar Protocolo B (SQL Editor) no Claude Code
   - [ ] Template de dispatch report comment
   - [ ] Teste end-to-end: `/dispatch-lovable` num Issue real
6. **Fase 3B — Agent Dispatch: Perplexity Pre/Post-flight:**
   - [ ] Pre-flight review integrado no `dispatch-prepare.yml`
   - [ ] Post-flight review no protocolo Claude Code
   - [ ] Modos por classificação (conservador/standard/rápido)
   - [ ] Teste: `/dispatch-lovable --with-review` com pre+post-flight
7. **Fase 3C — Agent Dispatch: Semi-automático (futuro):**
   - [ ] Auto-dispatch com countdown + opt-out
   - [ ] Telegram inline dispatch buttons
   - [ ] Dashboard custom (se justificado pelo volume de tasks)
