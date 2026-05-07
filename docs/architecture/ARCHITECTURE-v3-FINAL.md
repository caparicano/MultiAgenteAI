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
 └────────┘          └─────┬────┘   └──────────────────┘
                           │
              ┌────────────▼─────────────┐
              │  SUPABASE INTERNO        │
              │  (Lovable-managed)       │
              │                          │
              │  get2event, capi, hr     │
              │                          │
              │  ← SÓ Lovable escreve   │
              │  ← SÓ Luis cola SQL     │
              │    (permanentemente      │
              │     manual)              │
              └──────────────────────────┘

              ┌──────────────────────────────────────┐
              │  SUPABASE PESSOAL                    │
              │  (wqidcmapowfudwsazblj)              │
              │                                      │
              │  Usado APENAS para:                  │
              │  execute-migration (Fase 2)          │
              │  em projectos com Supabase EXTERNO   │
              │  (projectos futuros)                 │
              │                                      │
              │  NÃO usado para orquestração         │
              └──────────────────────────────────────┘
```

---

## Papéis

| Componente | Função |
|------------|--------|
| **Luis** | Gate final — aprova via GitHub Issues ou Telegram |
| **Claude Code** | Cria Issues (`gh issue create`), comenta planos, adiciona labels, commita ao repo |
| **Perplexity** | Chamado por GitHub Action → posta review como comment no Issue |
| **Lovable** | Lê `lovable/chat.md`, implementa, faz push de commits |
| **GitHub Actions** | Orquestra tudo: transitions, AI calls, Telegram, timeouts, CI |
| **GitHub Projects** | Dashboard visual (filtros nativos, board por estado) |
| **Supabase pessoal** | Só para execute-migration em projectos externos (Fase 2, futura) |

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
| Comando | Efeito |
|---------|--------|
| `/approve` | Aprova gate actual → transita para próximo estado |
| `/changes [texto]` | Pede alterações → volta ao estado anterior |
| `/migration-done` | Confirma SQL aplicado → vai para verify |
| `/verify-ok` | Confirma verificação → fecha Issue como done |
| `/block [motivo]` | Bloqueia manualmente |
| `/cancel` | Fecha Issue como cancelled |

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

---

## GitHub Actions (6 workflows)

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
     "lovable_chat_path": "lovable/chat.md",
     "orchestra_path": ".orchestra"
   }
   ```
3. Adicionar `orchestra.yml` ao repo do projecto (webhook → `notify-telegram.yml`)

**Configuração existente de projectos (Supabase interno):**
```
.orchestra/projects/get2event.json  → supabase_mode: "internal"
.orchestra/projects/capi.json       → supabase_mode: "internal"
.orchestra/projects/hr.json         → supabase_mode: "internal"
```

---

## Automação vs Manual

| Step | Automatizado? | Quem / Como |
|------|--------------|-------------|
| Criar task | Manual | Luis via `gh issue create` ou GitHub UI |
| Planning (plano + SQL) | Manual | Claude Code comenta no Issue |
| Perplexity AI review | **Auto** | `review-perplexity.yml` |
| **Gate 1 (plan review)** | **MANUAL** | Luis: `/approve` ou `/changes` no Issue |
| Gerar INBOX para Lovable | Manual | Claude Code comenta + commita `chat.md` |
| **Trigger Lovable** | **PERMANENTEMENTE MANUAL** | Luis diz "executa" no Lovable editor |
| CI (tsc, tests) | **Auto** | `ci.yml` |
| Scope drift check | **Auto** | `ci.yml` extensão (compara ficheiros planned vs PR) |
| **Gate 2 (code review)** | **MANUAL** | Luis: `/approve` ou `/changes` no Issue |
| Aplicar SQL (Supabase interno) | **PERMANENTEMENTE MANUAL** | Luis: copia SQL do comment → Lovable SQL Editor |
| Aplicar SQL (Supabase externo) | **Auto (Fase 2)** | `execute-migration.yml` |
| **Gate 3 (verify)** | **MANUAL** | Luis: `/verify-ok` no Issue |
| Gate timeouts | **Auto** | `gate-timeout.yml` (cron 15min) |
| Notificações Telegram | **Auto** | `notify-telegram.yml` (retry 3x backoff) |
| Cancelar task | Manual | Luis fecha o Issue |
| Audit log | **Auto** | GitHub Issue timeline (nativo, imutável) |

### Bridges irredutíveis (qualquer fase)
1. **Trigger Lovable** — Lovable não tem API
2. **Aplicar SQL em Supabase interno** — sem acesso externo ao Supabase do Lovable
3. **Gates humanos** — by design, Luis é o gate final

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

### Fase 3 — Polish

| # | Componente | Descrição |
|---|-----------|-----------|
| 12 | Issue templates | Templates no GitHub para criar tasks com estrutura correcta |
| 13 | `classify.yml` | Action que auto-classifica Issues novos via Claude API |
| 14 | `auto-inbox.yml` | Action que commita `lovable/chat.md` automaticamente |

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

---

## Custo

| Serviço | Custo |
|---------|-------|
| GitHub (repo público) | **€0** |
| GitHub Actions (repo público) | **€0 ilimitado** |
| GitHub Issues + Projects | **€0** |
| GitHub Secrets | **€0** |
| Perplexity API (~20 reviews/dia) | **~€3-5/mês** |
| Telegram Bot | **€0** |
| Supabase pessoal (só Fase 2+) | **€0** (já existe, activity mantém activo) |
| **Total** | **~€3-5/mês** |

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
