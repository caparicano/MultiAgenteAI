# MultiAgent AI — Plano de Arquitectura Consolidado (v2.2)

> Síntese das visões de Claude Code, Lovable e Perplexity + 2 rondas de revisão independente.
> Plataforma **multi-projecto**, **multi-sessão**, com Supabase **configurável** por projecto.
> **Status: DRAFT — pendente revisão humana**

---

## Changelog

### v2.1 → v2.2

| ID | Origem | Mudança |
|----|--------|---------|
| P1 | Perplexity Review #2 | Dashboard com **Supabase Auth** (magic link). RLS filtrada por `auth.uid()`. Sem acesso anónimo. |
| P2 | Perplexity Review #2 | Secrets documentados: `PERPLEXITY_API_KEY`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` no Vault |
| P3 | Perplexity Review #2 | Lock chat.md reforçado: GitHub Action detecta commits conflitantes |
| P4 | Perplexity Review #2 | **Fase 1 dividida em 1A + 1B** — MVP mínimo viável primeiro, depois dashboard e AI review |
| P5 | Perplexity Review #2 | `gate-timeout-check` via **Supabase Cron Jobs** (não pg_cron). Fallback: GitHub Actions scheduled |

### v2 → v2.1

| ID | Origem | Mudança |
|----|--------|---------|
| R1 | Perplexity Review #1 | Scope drift detection: `files_affected_actual` + comparação automática no Gate 2 |
| R2 | Perplexity Review #1 | Gate timeouts, prioridades, SLA. Campo `priority` em `orch_tasks`. Auto-block por timeout |
| R3 | Perplexity Review #1 | Dashboard usa anon key + RLS → **v2.2: substituído por Supabase Auth (P1)** |
| L1 | Perplexity Review #1 | Priorização multi-projecto nas notificações Telegram |
| L2 | Perplexity Review #1 | Perplexity review AI promovido para Fase 1B (chamada durante PLANNING → PLAN_REVIEW) |
| L3 | Perplexity Review #1 | Protocolo de lock em chat.md (STATUS header) + GitHub Action de conflito (P3) |
| L4 | Perplexity Review #1 | MVP testa caminhos de falha (rejeição, timeout, scope drift) |

---

## Context

O Luis gere múltiplos projectos (get2event, capi, hr, futuros) com 3 agentes AI (Claude Code, Lovable, Perplexity) coordenados manualmente. Problemas reais documentados:

- Lovable reporta "build clean" quando `tsc` falha (2026-05-04)
- Lovable renomeia `status` → `is_active` durante migração (schema drift)
- Agents fazem scope creep (pedido: 6 ficheiros, executado: 13+)
- Zero visibilidade cruzada — nenhum agente sabe o estado dos outros
- Luis é o router manual entre todas as plataformas
- **Cada projecto pode ter Supabase interno (Lovable) ou externo** — a plataforma tem de suportar ambos

**Objectivo:** Control Plane multi-projecto que orquestra o fluxo colaborativo, com dashboard visual, notificações Telegram, e audit trail completo.

---

## Constraint Arquitectural Crítico: Supabase Interno vs Externo

| Modo | Quem gere o Supabase | Control Plane pode conectar? | Migrations SQL |
|------|---------------------|------------------------------|----------------|
| **Interno (Lovable)** | Lovable — sem acesso externo | **NÃO** | **Permanentemente manuais** — Luis cola SQL no Lovable SQL Editor |
| **Externo (linked)** | Luis — service_role key disponível | **SIM** | Automatizáveis via `execute-migration` Edge Function |

**Projectos actuais:**
- `get2event` → Supabase **interno** (Lovable)
- `capi` → Supabase **interno** (Lovable)
- `hr` → Supabase **interno** (Lovable)

**Implicação:** Para os 3 projectos actuais, a migration SQL é um bridge **permanentemente manual**. O `execute-migration` automático só é possível em projectos futuros com Supabase externo.

---

## Decisões Arquitecturais (reconciliação das 3 visões + 2 revisões)

| Tensão | Claude Code | Lovable | Perplexity | **Decisão final** |
|--------|-------------|---------|------------|--------------------|
| Source of truth | Ficheiros no repo | Supabase Control Plane | — | **Ambos**: ficheiros para comunicar com Lovable, Supabase para tudo o resto |
| Orquestração | GitHub Actions + scripts | Edge Functions Deno | n8n | **Edge Functions** (sem infra extra). n8n adiado Fase 3 |
| Com. com Lovable | `lovable/chat.md` | Não abordou | Lock frágil (P3) | **`lovable/chat.md`** com lock protocol + GitHub Action de conflito |
| Bridges manuais | 3 irredutíveis | `execute_migration` resolve 1 | — | **Depende do modo Supabase**: interno = 3 bridges, externo = 2 bridges |
| Multi-projecto | Single-project | Single-project | — | **Multi-projecto** com `orch_projects` table + project selector no dashboard |
| Separação BD | 2 Supabase | 2 Supabase | — | **N+1 Supabase**: 1 Control Plane + N projectos alvo (cada um interno ou externo) |
| Auth dashboard | — | — | Zero auth (P1) | **Supabase Auth** magic link para luis.costa@get2c.pt |
| MVP scope | — | — | 10 componentes demais (P4) | **Fase 1A (motor) + 1B (UI + AI)** |

---

## Arquitectura

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                      LUIS (gate final)                            │
 │         Telegram (rotina) │ Dashboard (complexa) │ CLI (dev)     │
 │                           │ (auth: magic link)   │               │
 └──────────┬────────────────┴──────────────┬───────┴───────────────┘
            │                               │
 ┌──────────▼───────────────────────────────▼───────────────────────┐
 │           CONTROL PLANE SUPABASE (wqidcmapowfudwsazblj)         │
 │                                                                   │
 │  Auth: Supabase Auth (magic link — P1)                           │
 │    Allowed user: luis.costa@get2c.pt                              │
 │                                                                   │
 │  PostgreSQL:                                                      │
 │    orch_projects     (multi-projecto: config por projecto)        │
 │    orch_tasks        (FK → project, multi-sessão, priority)      │
 │    orch_plans        (versionados, FK → task)                    │
 │    orch_verdicts     (decisões por gate + AI reviews)            │
 │    orch_migrations   (SQL audit trail)                           │
 │    orch_audit_log    (append-only, imutável)                     │
 │    orch_rules        (classificação, config, gate timeouts)      │
 │                                                                   │
 │  RLS Policies (v2.2 — P1):                                       │
 │    authenticated: SELECT all | INSERT verdicts | UPDATE tasks    │
 │    service_role: Edge Functions only (Vault secrets)              │
 │    anon: ZERO access                                              │
 │                                                                   │
 │  Vault Secrets (v2.2 — P2):                                      │
 │    PERPLEXITY_API_KEY    (sonar-reasoning-pro)                   │
 │    TELEGRAM_BOT_TOKEN    (Bot API)                                │
 │    TELEGRAM_CHAT_ID      (Luis chat)                              │
 │    [per-project] SERVICE_ROLE_KEY_{project_id} (só externos)     │
 │                                                                   │
 │  Edge Functions:                                                  │
 │    orchestrate         (state machine, project-aware)             │
 │    notify-telegram     (push + inline approve/reject + priority) │
 │    github-webhook      (CI status, multi-repo)                   │
 │    review-perplexity   (AI review — Fase 1B)                     │
 │    gate-timeout-check  (Supabase Cron Jobs — P5)                 │
 │    execute-migration   (só Supabase EXTERNO — Fase 2)            │
 │    ingest-verdict      (Telegram callback — Fase 2)              │
 │    classify            (auto-classificação — Fase 2)             │
 │                                                                   │
 │  Dashboard SPA (React+Vite+shadcn) — Supabase Auth + RLS:       │
 │    Login (magic link) → Project selector → Task list             │
 │    → Task detail (scope diff) → Audit log                        │
 └───┬──────────┬──────────────┬──────────────┬─────────────────────┘
     │          │              │              │
     ▼          ▼              ▼              ▼
 ┌────────┐ ┌────────┐  ┌──────────┐  ┌──────────────┐
 │ Claude │ │Perplex.│  │ Lovable  │  │   GitHub     │
 │ Code   │ │sonar-  │  │ (lê/     │  │ (N repos:    │
 │        │ │reason. │  │  escreve  │  │  get2event,  │
 │ progra-│ │-pro    │  │  chat.md  │  │  capi, hr,   │
 │ mador  │ │        │  │  + push)  │  │  ...)        │
 │        │ │revisor │  │          │  │              │
 └────────┘ └────────┘  └─────┬────┘  └──────┬───────┘
                              │               │
                   ┌──────────┴───────────────┴──────────┐
                   │                                      │
          ┌────────▼─────────┐              ┌─────────────▼──────────┐
          │ SUPABASE INTERNO │              │  SUPABASE EXTERNO      │
          │ (Lovable-managed)│              │  (Luis-managed)        │
          │                  │              │                        │
          │ get2event        │              │ (projectos futuros)    │
          │ capi             │              │                        │
          │ hr               │              │ ← execute-migration    │
          │                  │              │   pode conectar via    │
          │ ← SÓ Lovable    │              │   service_role key     │
          │   pode escrever  │              │                        │
          └──────────────────┘              └────────────────────────┘
```

---

## Papéis

| Componente | Função | Pode escrever na BD do projecto? |
|------------|--------|----------------------------------|
| **Luis** | Gate final, override, instruções | Indirectamente (approve) ou manual (SQL editor para Supabase interno) |
| **Claude Code** | Programador: plano + SQL + código + reviews | Não (commits no repo, não na BD) |
| **Perplexity** | Revisor #2: verdicts + risks + citações (activo desde Fase 1B — L2) | Não |
| **Lovable** | Implementador: código + SQL no seu Supabase interno | Sim (Supabase interno). Não (Supabase externo) |
| **Control Plane** | Orquestração, audit, secrets, notificações | Só Supabase externo (via service_role key) |

---

## State Machine (7 estados)

```
INTAKE ──→ PLANNING ──→ PLAN_REVIEW ──→ EXECUTION ──→ CODE_REVIEW ──→ MIGRATION ──→ VERIFY ──→ DONE
                  │      ▲ GATE 1       (pode loop    ▲ GATE 2       (se SQL)      ▲ GATE 3
                  │      │ (humano)      com Lovable)  │ (humano)                   │ (humano)
                  │      │               + scope diff  │                            │
                  │      │                (R1)         │                            │
                  │      └── changes_requested ────────┘                            │
                  │                                                                 │
                  │      └──────────── blocked (timeout R2) / cancelled ────────────┘
                  │
                  └──→ Perplexity AI review (L2, Fase 1B)
                       Resultado gravado em orch_verdicts
                       antes de PLAN_REVIEW
```

**Nota:** O estado MIGRATION comporta-se diferentemente conforme o modo Supabase:
- **Interno:** Control Plane gera SQL, notifica Luis, Luis cola no Lovable SQL Editor, Luis confirma → VERIFY
- **Externo:** Control Plane executa SQL via `execute-migration` (com 5 camadas defesa) → VERIFY

### Gate Timeouts (R2)

| Classificação | Gate timeout | Acção ao expirar |
|--------------|-------------|-------------------|
| **Rotina** | 4 horas | Auto-transita para `blocked` + notifica Telegram |
| **Complexa** | 24 horas | Auto-transita para `blocked` + notifica Telegram |
| **Crítica** | 72 horas | Auto-transita para `blocked` + notifica Telegram + email |

Implementação: **Supabase Cron Jobs** (v2.2 — P5). A Edge Function `gate-timeout-check` é invocada via Supabase Cron a cada 15 minutos. Se Supabase Cron não estiver disponível no free tier, fallback para **GitHub Actions scheduled** (`cron: '*/15 * * * *'`) que faz POST ao webhook.

### Classificação (determina canais + automação)

| Tipo | Critério | Canal primário | Gates automáticos |
|------|----------|---------------|-------------------|
| **Rotina** | Sem SQL, single-agent | Telegram push + botão inline | Gates 2-3 podem ser auto se CI ✅ |
| **Complexa** | Envolve Lovable ou SQL | Dashboard + Telegram | Todos manuais |
| **Crítica** | RLS, auth, drop, alter sensível | Dashboard + email | Todos manuais + confirmação dupla |

---

## Data Model (7 tabelas, prefixo `orch_`)

No Supabase pessoal (wqidcmapowfudwsazblj), isolado das tabelas existentes.

### `orch_projects`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| name | TEXT NOT NULL UNIQUE | Ex: "get2event", "capi", "hr" |
| github_repo | TEXT NOT NULL | Ex: "tech2c-devteam/get2event" |
| supabase_mode | TEXT NOT NULL CHECK (supabase_mode IN ('internal','external')) | |
| supabase_project_id | TEXT | ID do Supabase (se externo) |
| supabase_url | TEXT | URL (se externo) |
| lovable_chat_path | TEXT DEFAULT 'lovable/chat.md' | Caminho do INBOX/OUTBOX no repo |
| orchestra_path | TEXT DEFAULT '.orchestra' | Caminho do state.json no repo |
| active | BOOLEAN DEFAULT true | |
| created_at | TIMESTAMPTZ DEFAULT now() | |

### `orch_tasks`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK → orch_projects NOT NULL | |
| title | TEXT NOT NULL | |
| description | TEXT | |
| source | TEXT DEFAULT 'manual' CHECK (source IN ('manual','telegram','github_issue')) | |
| classification | TEXT DEFAULT 'routine' CHECK (classification IN ('routine','complex','critical')) | |
| priority | TEXT DEFAULT 'medium' CHECK (priority IN ('low','medium','high','critical')) | R2 |
| status | TEXT DEFAULT 'intake' | intake / planning / plan_review / execution / code_review / migration / verify / done / blocked / cancelled |
| blocked_reason | TEXT | R2: gate_timeout, scope_drift, changes_requested |
| target_branch | TEXT | |
| pr_number | INTEGER | |
| assigned_agents | JSONB DEFAULT '[]' | |
| files_affected_planned | JSONB DEFAULT '[]' | R1 |
| files_affected_actual | JSONB DEFAULT '[]' | R1 |
| metadata | JSONB DEFAULT '{}' | |
| created_at | TIMESTAMPTZ DEFAULT now() | |
| updated_at | TIMESTAMPTZ DEFAULT now() | |

### `orch_plans`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK → orch_tasks NOT NULL | |
| version | INTEGER NOT NULL DEFAULT 1 | |
| content | TEXT NOT NULL | Markdown do plano |
| sql_draft | TEXT | SQL migration (se aplicável) |
| lovable_inbox | TEXT | Draft da mensagem INBOX para chat.md |
| files_affected | JSONB DEFAULT '[]' | |
| author | TEXT NOT NULL DEFAULT 'claude_code' | |
| created_at | TIMESTAMPTZ DEFAULT now() | |
| UNIQUE(task_id, version) | | |

### `orch_verdicts`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK → orch_tasks NOT NULL | |
| gate | TEXT NOT NULL CHECK (gate IN ('plan_review','code_review','verify')) | |
| decision | TEXT NOT NULL CHECK (decision IN ('approved','changes_requested','rejected')) | |
| reviewer | TEXT NOT NULL | 'luis' / 'perplexity' / 'claude_code' |
| comments | TEXT | |
| ai_score | INTEGER CHECK (ai_score BETWEEN 1 AND 10) | |
| ai_findings | JSONB | |
| scope_diff | JSONB | R1: {planned, actual, extra, missing} |
| plan_version | INTEGER | |
| created_at | TIMESTAMPTZ DEFAULT now() | |

### `orch_migrations`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK → orch_tasks NOT NULL | |
| sql_content | TEXT NOT NULL | |
| execution_mode | TEXT NOT NULL CHECK (execution_mode IN ('manual','automatic')) | |
| status | TEXT DEFAULT 'pending' | pending / pre_gate_ok / applied / post_gate_ok / done / failed / rolled_back |
| pre_gate_result | JSONB | |
| post_gate_result | JSONB | |
| error_message | TEXT | |
| applied_by | TEXT | 'luis' (manual) ou 'execute-migration' (auto) |
| applied_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ DEFAULT now() | |

### `orch_audit_log` (append-only)

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK → orch_projects | |
| task_id | UUID FK → orch_tasks | |
| actor | TEXT NOT NULL | claude_code / lovable / perplexity / luis / system |
| action | TEXT NOT NULL | state_transition / commit / plan_written / verdict / migration_applied / notification_sent / gate_timeout / scope_drift_detected |
| from_state | TEXT | |
| to_state | TEXT | |
| details | JSONB DEFAULT '{}' | |
| created_at | TIMESTAMPTZ DEFAULT now() | |

Trigger de imutabilidade:
```sql
CREATE OR REPLACE FUNCTION orch_audit_log_immutable()
RETURNS TRIGGER AS $$
BEGIN
  RAISE EXCEPTION 'orch_audit_log is append-only: UPDATE and DELETE are forbidden';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_audit_immutability
  BEFORE UPDATE OR DELETE ON orch_audit_log
  FOR EACH ROW EXECUTE FUNCTION orch_audit_log_immutable();
```

### `orch_rules`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK → orch_projects (nullable) | NULL = regra global |
| rule_key | TEXT NOT NULL | |
| rule_value | JSONB NOT NULL | |
| description | TEXT | |
| active | BOOLEAN DEFAULT true | |
| created_at | TIMESTAMPTZ DEFAULT now() | |
| updated_at | TIMESTAMPTZ DEFAULT now() | |
| UNIQUE(project_id, rule_key) | | |

---

## RLS Policies (v2.2 — P1)

O Dashboard requer **Supabase Auth login** (magic link para `luis.costa@get2c.pt`). Sem login, zero acesso. A `service_role_key` fica **exclusivamente** nas Edge Functions (Vault secrets).

```sql
-- Enable RLS em todas as tabelas orch_*
ALTER TABLE orch_projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_verdicts ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_migrations ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_audit_log ENABLE ROW LEVEL SECURITY;
ALTER TABLE orch_rules ENABLE ROW LEVEL SECURITY;

-- Leitura: apenas utilizadores autenticados
CREATE POLICY "auth_read" ON orch_projects FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_tasks FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_plans FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_verdicts FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_migrations FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_audit_log FOR SELECT TO authenticated USING (true);
CREATE POLICY "auth_read" ON orch_rules FOR SELECT TO authenticated USING (true);

-- Escrita via dashboard: apenas verdicts (gates do Luis)
CREATE POLICY "auth_insert_verdicts" ON orch_verdicts FOR INSERT TO authenticated
  WITH CHECK (reviewer = 'luis');

-- Update: Luis pode actualizar status de tasks (ex: confirmar migration manual)
CREATE POLICY "auth_update_tasks" ON orch_tasks FOR UPDATE TO authenticated
  USING (true) WITH CHECK (true);

-- Tudo o resto (INSERT plans, migrations, audit, rules) = service_role only (Edge Functions)
-- Sem policy = negado por defeito com RLS enabled

-- ZERO acesso anon
-- (nenhuma policy para anon = bloqueio total)
```

---

## Vault Secrets (v2.2 — P2)

Todos os secrets ficam no **Supabase Vault** do Control Plane, acessíveis apenas por Edge Functions.

| Secret | Usado por | Descrição |
|--------|-----------|-----------|
| `PERPLEXITY_API_KEY` | `review-perplexity` | API key para `sonar-reasoning-pro` |
| `TELEGRAM_BOT_TOKEN` | `notify-telegram`, `ingest-verdict` | Token do Telegram Bot |
| `TELEGRAM_CHAT_ID` | `notify-telegram` | ID do chat do Luis |
| `SERVICE_ROLE_KEY_{project_id}` | `execute-migration` | Só para projectos com `supabase_mode: 'external'`. Um secret por projecto externo. |

**Provisionamento:**
1. Luis cria secrets via Supabase Dashboard → Settings → Vault
2. Edge Functions acedem via `Deno.env.get('SECRET_NAME')` (secrets expostos como env vars nas Edge Functions)
3. Nunca em código, nunca no frontend, nunca em `.env` commitado

---

## Protocolo de Comunicação

### Com Lovable (ficheiros + lock protocol — L3 + P3)

**Protocolo de lock:**
```markdown
## STATUS: CLEAR | INBOX_PENDING | OUTBOX_READY

Regras:
- Claude Code só escreve INBOX se STATUS = CLEAR
- Após escrever INBOX, Claude Code muda STATUS → INBOX_PENDING
- Lovable só responde se STATUS = INBOX_PENDING
- Após responder, Lovable muda STATUS → OUTBOX_READY
- Após ler OUTBOX, Claude Code muda STATUS → CLEAR
- Qualquer outra combinação = CONFLITO → notifica Luis
```

**Detecção de conflito via GitHub Action (v2.2 — P3):**
```yaml
# .github/workflows/chatmd-conflict-check.yml
# Trigger: on push que altera lovable/chat.md
# Lógica:
#   1. Lê STATUS header do chat.md
#   2. Verifica quem fez o commit (Claude Code vs Lovable)
#   3. Se Claude commita e STATUS != CLEAR → conflito
#   4. Se Lovable commita e STATUS != INBOX_PENDING → conflito
#   5. Se conflito → POST para notify-telegram Edge Function
```

Ficheiros no repo:
```
{project.lovable_chat_path}              ← INBOX/OUTBOX (com STATUS header)
{project.orchestra_path}/state.json      ← Estado machine-readable
{project.orchestra_path}/current-plan.md ← Plano actual
```

### Com Claude Code (MCP + gh CLI)
- Supabase MCP: `execute_sql` para ler/escrever orch_tables no Control Plane
- GitHub: `gh pr/issue/api` para operações em qualquer repo registado
- HTTP: `fetch()` para Perplexity API

### Com Perplexity (API REST) — activo desde Fase 1B (L2)
```
POST api.perplexity.ai/chat/completions
  model: "sonar-reasoning-pro"
  system: "You are a code reviewer for a Supabase+React project.
           Return JSON: {score: 1-10, findings: [{severity, description, recommendation}], approved: bool}"
  user: [conteúdo do plano/diff/SQL + contexto do projecto]
```

### Dashboard ← Control Plane (Supabase Realtime)
Subscriptions filtradas por `project_id` nas tabelas `orch_tasks`, `orch_verdicts`, `orch_audit_log`. Requerem token de autenticação (magic link session).

---

## Scope Drift Detection (R1)

### Fluxo no Gate 2 (CODE_REVIEW)

1. Claude Code (ou GitHub Action) analisa o PR/diff do Lovable
2. Extrai lista de ficheiros alterados → grava em `orch_tasks.files_affected_actual`
3. Compara com `orch_tasks.files_affected_planned` (copiado do plano)
4. Se existem ficheiros extra (não planeados):
   - Verifica regra `lovable_scope_drift_tolerance` em `orch_rules`
   - Se `max_extra_files: 0` e há extras → **bloqueia Gate 2**
   - Grava scope diff em `orch_verdicts.scope_diff`
   - Notifica Luis via Telegram com lista de ficheiros extra
5. Luis decide: aprovar com override, ou rejeitar

---

## Defesas antes de tocar BD do projecto alvo

### Supabase EXTERNO (automático, 5 camadas)
1. Plano `approved` em `orch_verdicts`
2. Verdicts AI gravados (existem, mesmo se divergentes)
3. Gate queries pré-execução validam pré-condições
4. SQL em transacção (`BEGIN…COMMIT/ROLLBACK`)
5. Gate queries pós-execução validam efeito
6. Audit log imutável

### Supabase INTERNO (manual, 3 camadas)
1. Plano `approved` em `orch_verdicts`
2. Verdicts AI gravados
3. SQL gerado + gate queries disponíveis no dashboard para Luis executar manualmente
4. Luis confirma resultado → estado transita
5. Audit log imutável

---

## Componentes a Construir (v2.2 — faseamento revisto P4)

### Fase 1A — Motor mínimo (~1-2 weekends)

Objectivo: **uma task percorre os 7 estados end-to-end**, com audit trail e notificações Telegram. Sem dashboard, sem AI review.

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | `orch_*` schema | SQL migration | 7 tabelas + RLS (authenticated) + trigger audit imutável + CHECK constraints |
| 2 | Seed projectos + regras | SQL | get2event, capi, hr (`internal`) + regras timeout/scope |
| 3 | `.orchestra/state.json` | Ficheiro repo | `{project_id, task_id, status, plan_version, branch}` |
| 4 | `orchestrate` Edge Function | Deno | State machine project-aware: transições, scope drift check, audit log |
| 5 | `notify-telegram` Edge Function | Deno | Push + inline buttons + priority badge + wait time |
| 6 | `github-webhook` Edge Function | Deno | CI status multi-repo |
| 7 | `orchestra.yml` GitHub Action | YAML | Template reutilizável por repo |

**Critério de saída Fase 1A:** 1 task real (ex: hotfix get2event) percorre INTAKE → DONE via CLI + Telegram, com audit log completo.

### Fase 1B — UI + AI review (~1-2 weekends)

Objectivo: **Luis vê tudo no dashboard e tem review AI antes de aprovar**.

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 8 | `review-perplexity` Edge Function | Deno | AI review automático (PLANNING → PLAN_REVIEW) |
| 9 | `gate-timeout-check` | Supabase Cron / GH Actions | Scheduled 15min, auto-block tasks expiradas |
| 10 | `chatmd-conflict-check.yml` | GitHub Action | Detecta conflitos no chat.md (P3) |
| 11 | Dashboard v0 | React SPA (Lovable) | 5 páginas: login, project selector, task list (sorted priority + wait), task detail (scope diff + AI review), audit log |

**Critério de saída Fase 1B:** Dashboard funcional com login magic link. Luis vê score Perplexity antes de aprovar. Tasks em timeout são auto-blocked.

### Fase 2 — Automação (~2-3 weekends)

| # | Componente | Descrição |
|---|-----------|-----------|
| 12 | `execute-migration` Edge Function | SQL gated — só `supabase_mode = 'external'` |
| 13 | `ingest-verdict` Edge Function | Telegram callback → orch_verdicts (inline button approval sem abrir dashboard) |
| 14 | `classify` Edge Function | Auto-classificação via Claude API |
| 15 | Auto-commit chat.md | `orchestrate` commita INBOX via GitHub API (com lock protocol) |

### Fase 3 — Polish (ongoing)

| # | Componente | Descrição |
|---|-----------|-----------|
| 16 | Schema diff view | Dashboard mostra SQL planeado vs schema real |
| 17 | Manual migration assistant | SQL formatado + gate queries + checklist para Lovable |
| 18 | n8n integration | Visual workflow builder (opcional) |
| 19 | PWA/mobile | Dashboard installable |

---

## Automatizado vs Manual

### Tasks sem SQL (igual para ambos os modos)

| Step | Fase 1A | Fase 1B | Fase 2 |
|------|---------|---------|--------|
| Task creation | Manual (CLI/Telegram) | Manual (dashboard) | Manual (dashboard) |
| Classification | Manual | Manual | Auto (classify) |
| Planning | Auto (Claude Code) | Auto (Claude Code) | Auto |
| Perplexity AI review | — | Auto (review-perplexity) | Auto |
| **Plan review (Gate 1)** | **Manual (Telegram)** | **Manual (dashboard, com score AI)** | **Manual** |
| Code writing | Auto (Claude Code) | Auto | Auto |
| Lovable INBOX | Manual | Manual | Auto (auto-commit) |
| **Trigger Lovable** | **MANUAL** | **MANUAL** | **MANUAL (permanente)** |
| CI | Auto (Actions) | Auto | Auto |
| Scope drift check | — | Auto (orchestrate) | Auto |
| **Code review (Gate 2)** | **Manual (Telegram)** | **Manual (dashboard, scope diff)** | **Manual** |
| **Verify (Gate 3)** | **Manual (Telegram)** | **Manual (dashboard)** | **Manual** |
| Gate timeout | — | Auto (cron) | Auto |

### Tasks com SQL — modo INTERNO

| Step | Auto? | Quem? |
|------|-------|-------|
| SQL gerado | Auto | Claude Code |
| Gate queries geradas | Auto | Claude Code |
| **SQL colado no Lovable SQL Editor** | **PERMANENTEMENTE MANUAL** | **Luis** |
| **Gate queries executadas** | **MANUAL** | **Luis** |
| **Confirmação** | **GATE 3** | **Luis** |

### Tasks com SQL — modo EXTERNO (Fase 2+)

| Step | Auto? | Quem? |
|------|-------|-------|
| SQL + gates | Auto | execute-migration (transacção + 5 camadas) |
| **Verify (Gate 3)** | **Manual** | **Luis** |

### Bridges irredutíveis (qualquer modo, qualquer fase)
1. **Trigger Lovable** — sem API
2. **Aplicar SQL em Supabase interno** — sem acesso externo
3. **Gates humanos** — by design

---

## Tech Stack

| Camada | Escolha | Justificação |
|--------|---------|-------------|
| Control Plane DB | Supabase pessoal (wqidcmapowfudwsazblj) | Já provisionado, MCP configurado, free tier |
| Auth | Supabase Auth magic link (P1) | Zero password, single user, já incluído no Supabase |
| Orchestration | Edge Functions (Deno) | Sem infra extra |
| CI | GitHub Actions | Já existe em todos os repos |
| Notificações | Telegram Bot API | Free, instant, inline keyboards |
| Scheduled jobs | Supabase Cron Jobs (P5) | Fallback: GitHub Actions scheduled |
| Dashboard | Lovable (React+Vite+shadcn) no Control Plane Supabase | Build rápido |
| Com. com Lovable | `lovable/chat.md` + lock protocol (L3 + P3) | Único canal possível |
| Estado no repo | `.orchestra/state.json` por projecto | Legível por todos os agents |
| AI Review | Perplexity `sonar-reasoning-pro` | Desde Fase 1B |
| Cross-project SQL | service_role key (só modo externo) | Vault secrets |
| NÃO n8n | Adiado Fase 3 | Edge Functions suficientes |

---

## Verificação

### Fase 1A — Motor mínimo

**Happy path (CLI + Telegram):**
1. `execute_sql` → insert task em `orch_tasks` (project: get2event, priority: high)
2. POST `orchestrate` → transição intake → planning
3. `execute_sql` → insert plan em `orch_plans`
4. POST `orchestrate` → transição planning → plan_review → Telegram recebe notificação com priority badge
5. POST `orchestrate` → verdict approved → transição plan_review → execution
6. (simular) → transição até done
7. Verificar `orch_audit_log` tem todas as transições

**Rejeição:**
8. Task em plan_review → verdict `changes_requested` → volta a planning com `blocked_reason`

**Multi-projecto:**
9. Repetir com task em capi — confirmar isolamento de audit logs

### Fase 1B — Dashboard + AI

**Happy path (Dashboard):**
10. Login magic link (luis.costa@get2c.pt) → ver project selector
11. Criar task via dashboard → ver na task list sorted by priority
12. Perplexity review automático → ver score + findings no task detail
13. Aprovar via dashboard → estado transita

**Timeout:**
14. Task rotina em plan_review → esperar (ou simular) 4h → auto-blocked + notificação Telegram

**Scope drift:**
15. Task com `files_affected_actual` diferente de `planned` → Gate 2 bloqueado → scope diff visível no dashboard

**Critério de sucesso global:** Tasks em 2 projectos percorrem ciclos felizes e de falha simultaneamente, com audit trails separados, notificações correctas, e dashboard a mostrar tudo em real-time.

---

## Próximos Passos

1. ⏳ Luis revê arquitectura v2.2
2. ✅ Repo GitHub criado (https://github.com/caparicano/MultiAgenteAI)
3. **Fase 1A:**
   - [ ] Aplicar migration `orch_*` (7 tabelas + RLS + triggers) no Supabase pessoal via MCP
   - [ ] Seed: get2event, capi, hr + regras
   - [ ] Provisionar Vault secrets: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`
   - [ ] Criar Telegram Bot via @BotFather
   - [ ] Edge Function: `orchestrate` (state machine)
   - [ ] Edge Function: `notify-telegram`
   - [ ] Edge Function: `github-webhook`
   - [ ] GitHub Action: `orchestra.yml` (template)
   - [ ] Teste end-to-end via CLI + Telegram
4. **Fase 1B:**
   - [ ] Provisionar Vault secret: `PERPLEXITY_API_KEY`
   - [ ] Edge Function: `review-perplexity`
   - [ ] Edge Function / Cron: `gate-timeout-check`
   - [ ] GitHub Action: `chatmd-conflict-check.yml`
   - [ ] Dashboard v0 (Lovable no Control Plane): login + project selector + task list + task detail + audit log
   - [ ] Teste end-to-end via Dashboard
