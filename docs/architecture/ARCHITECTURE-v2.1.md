# MultiAgent AI — Plano de Arquitectura Consolidado (v2.1)

> Síntese das visões de Claude Code, Lovable e Perplexity + clarificações do Luis.
> Inclui findings da revisão independente do Perplexity (7 Mai 2026, commit a3e754f).
> Plataforma **multi-projecto**, **multi-sessão**, com Supabase **configurável** por projecto.
> **Status: DRAFT — pendente revisão humana**

---

## Changelog v2 → v2.1

| ID | Origem | Mudança |
|----|--------|---------|
| R1 | Perplexity — Risco Alto 1 | Scope drift detection: `files_affected_actual` + comparação automática no Gate 2 |
| R2 | Perplexity — Risco Alto 2 | Gate timeouts, prioridades, SLA. Campo `priority` em `orch_tasks`. Auto-block por timeout |
| R3 | Perplexity — Risco Alto 3 | Dashboard usa **anon key + RLS**, nunca service_role. Políticas RLS explícitas para `orch_*` |
| L1 | Perplexity — Lacuna 1 | Priorização multi-projecto nas notificações Telegram |
| L2 | Perplexity — Lacuna 2 | Perplexity review AI promovido para Fase 1 (chamada durante PLANNING → PLAN_REVIEW) |
| L3 | Perplexity — Lacuna 3 | Protocolo de lock em `chat.md` (STATUS: CLEAR / INBOX_PENDING / OUTBOX_READY) |
| L4 | Perplexity — Lacuna 4 | MVP testa também caminhos de falha (rejeição, timeout, blocked) |

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

## Decisões Arquitecturais (reconciliação das 3 visões)

| Tensão | Claude Code | Lovable | Perplexity | **Decisão final** |
|--------|-------------|---------|------------|--------------------|
| Source of truth | Ficheiros no repo | Supabase Control Plane | — | **Ambos**: ficheiros para comunicar com Lovable, Supabase para tudo o resto |
| Orquestração | GitHub Actions + scripts | Edge Functions Deno | n8n | **Edge Functions** (sem infra extra). n8n adiado Fase 3 |
| Com. com Lovable | `lovable/chat.md` | Não abordou | — | **`lovable/chat.md`** (Lovable tem ZERO API) |
| Bridges manuais | 3 irredutíveis | `execute_migration` resolve 1 | — | **Depende do modo Supabase**: interno = 3 bridges, externo = 2 bridges |
| Multi-projecto | Single-project | Single-project | — | **Multi-projecto** com `orch_projects` table + project selector no dashboard |
| Separação BD | 2 Supabase | 2 Supabase | — | **N+1 Supabase**: 1 Control Plane + N projectos alvo (cada um interno ou externo) |

---

## Arquitectura

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                      LUIS (gate final)                            │
 │         Telegram (rotina) │ Dashboard (complexa) │ CLI (dev)     │
 └──────────┬────────────────┴──────────────┬───────┴───────────────┘
            │                               │
 ┌──────────▼───────────────────────────────▼───────────────────────┐
 │           CONTROL PLANE SUPABASE (wqidcmapowfudwsazblj)         │
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
 │  RLS Policies (v2.1 — R3):                                       │
 │    anon: SELECT all orch_* | INSERT orch_verdicts only           │
 │    service_role: Edge Functions only (Vault secrets)              │
 │                                                                   │
 │  Edge Functions:                                                  │
 │    orchestrate         (state machine, project-aware)             │
 │    notify-telegram     (push + inline approve/reject + priority) │
 │    github-webhook      (CI status, multi-repo)                   │
 │    review-perplexity   (v2.1 — L2: chamada API na Fase 1)       │
 │    execute-migration   (só projectos com Supabase EXTERNO)       │
 │    ingest-verdict      (Telegram callback)                       │
 │    classify            (auto-classificação)                      │
 │    gate-timeout-check  (v2.1 — R2: cron/scheduled)              │
 │                                                                   │
 │  Dashboard SPA (React+Vite+shadcn) — usa anon key + RLS:        │
 │    Project selector → Task list (sorted by priority + wait time) │
 │    → Task detail (scope diff view) → Audit log                   │
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
| **Perplexity** | Revisor #2: verdicts + risks + citações (activo desde Fase 1 — L2) | Não |
| **Lovable** | Implementador: código + SQL no seu Supabase interno | Sim (Supabase interno). Não (Supabase externo — não tem acesso) |
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
                  └──→ Perplexity AI review (L2)
                       Resultado gravado em orch_verdicts
                       antes de PLAN_REVIEW
```

**Nota:** O estado MIGRATION comporta-se diferentemente conforme o modo Supabase:
- **Interno:** Control Plane gera SQL, notifica Luis, Luis cola no Lovable SQL Editor, Luis confirma → VERIFY
- **Externo:** Control Plane executa SQL via `execute-migration` (com 5 camadas defesa) → VERIFY

### Gate Timeouts (v2.1 — R2)

| Classificação | Gate timeout | Acção ao expirar |
|--------------|-------------|-------------------|
| **Rotina** | 4 horas | Auto-transita para `blocked` + notifica Telegram |
| **Complexa** | 24 horas | Auto-transita para `blocked` + notifica Telegram |
| **Crítica** | 72 horas | Auto-transita para `blocked` + notifica Telegram + email |

A Edge Function `gate-timeout-check` corre periodicamente (ex: cada 15 min via pg_cron ou Supabase scheduled function) e verifica tasks cujo `updated_at` + timeout < now().

### Classificação (determina canais + automação)

| Tipo | Critério | Canal primário | Gates automáticos |
|------|----------|---------------|-------------------|
| **Rotina** | Sem SQL, single-agent | Telegram push + botão inline | Gates 2-3 podem ser auto se CI ✅ |
| **Complexa** | Envolve Lovable ou SQL | Dashboard + Telegram | Todos manuais |
| **Crítica** | RLS, auth, drop, alter sensível | Dashboard + email | Todos manuais + confirmação dupla |

---

## Data Model (7 tabelas, prefixo `orch_`)

No Supabase pessoal (wqidcmapowfudwsazblj), isolado das tabelas existentes.

### `orch_projects` (multi-projecto)

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| name | TEXT NOT NULL | Ex: "get2event", "capi", "hr" |
| github_repo | TEXT NOT NULL | Ex: "tech2c-devteam/get2event" |
| supabase_mode | TEXT NOT NULL | `'internal'` ou `'external'` |
| supabase_project_id | TEXT | ID do Supabase (se externo) |
| supabase_url | TEXT | URL (se externo) |
| lovable_chat_path | TEXT DEFAULT 'lovable/chat.md' | Caminho do INBOX/OUTBOX no repo |
| orchestra_path | TEXT DEFAULT '.orchestra' | Caminho do state.json no repo |
| active | BOOLEAN DEFAULT true | |
| created_at | TIMESTAMPTZ | |

**Nota:** `service_role_key` NÃO fica nesta tabela — fica nos Vault secrets do Supabase, referenciado por `supabase_project_id`.

### `orch_tasks` (FK → project) — actualizado v2.1

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| **project_id** | UUID FK → orch_projects | **Qual projecto** |
| title | TEXT NOT NULL | |
| description | TEXT | |
| source | TEXT DEFAULT 'manual' | manual / telegram / github_issue |
| classification | TEXT DEFAULT 'routine' | routine / complex / critical |
| **priority** | TEXT DEFAULT 'medium' | **v2.1 R2:** low / medium / high / critical |
| status | TEXT DEFAULT 'intake' | intake → planning → plan_review → execution → code_review → migration → verify → done / blocked / cancelled |
| **blocked_reason** | TEXT | **v2.1 R2:** Ex: "gate_timeout", "scope_drift", "changes_requested" |
| target_branch | TEXT | |
| pr_number | INTEGER | |
| assigned_agents | JSONB DEFAULT '[]' | ["claude_code","lovable","perplexity"] |
| **files_affected_planned** | JSONB DEFAULT '[]' | **v2.1 R1:** Ficheiros do plano (do orch_plans) |
| **files_affected_actual** | JSONB DEFAULT '[]' | **v2.1 R1:** Ficheiros reais (do PR/diff do Lovable) |
| metadata | JSONB DEFAULT '{}' | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

### `orch_plans`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK → orch_tasks | |
| version | INTEGER DEFAULT 1 | |
| content | TEXT NOT NULL | Markdown do plano |
| sql_draft | TEXT | SQL migration (se aplicável) |
| lovable_inbox | TEXT | Draft da mensagem INBOX para chat.md |
| files_affected | JSONB DEFAULT '[]' | |
| author | TEXT DEFAULT 'claude_code' | |
| created_at | TIMESTAMPTZ | |

### `orch_verdicts` — actualizado v2.1

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK | |
| gate | TEXT NOT NULL | plan_review / code_review / verify |
| decision | TEXT NOT NULL | approved / changes_requested / rejected |
| reviewer | TEXT NOT NULL | 'luis' / 'perplexity' / 'claude_code' |
| comments | TEXT | |
| ai_score | INTEGER | Score 1-10 (se review AI) |
| ai_findings | JSONB | Findings estruturados (se review AI) |
| **scope_diff** | JSONB | **v2.1 R1:** `{planned: [...], actual: [...], extra: [...], missing: [...]}` |
| plan_version | INTEGER | |
| created_at | TIMESTAMPTZ | |

### `orch_migrations`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK | |
| sql_content | TEXT NOT NULL | |
| **execution_mode** | TEXT NOT NULL | `'manual'` (Supabase interno) ou `'automatic'` (Supabase externo) |
| status | TEXT DEFAULT 'pending' | pending → pre_gate_ok → applied → post_gate_ok → done / failed / rolled_back |
| pre_gate_result | JSONB | |
| post_gate_result | JSONB | |
| error_message | TEXT | |
| applied_by | TEXT | 'luis' (manual) ou 'execute-migration' (auto) |
| applied_at | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ | |

### `orch_audit_log` (append-only, trigger bloqueia UPDATE/DELETE)

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK | |
| task_id | UUID FK | |
| actor | TEXT NOT NULL | claude_code / lovable / perplexity / luis / system |
| action | TEXT NOT NULL | state_transition / commit / plan_written / verdict / migration_applied / notification_sent / **gate_timeout** / **scope_drift_detected** |
| from_state | TEXT | |
| to_state | TEXT | |
| details | JSONB DEFAULT '{}' | |
| created_at | TIMESTAMPTZ | |

### `orch_rules` — actualizado v2.1

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK (nullable) | NULL = regra global |
| rule_key | TEXT NOT NULL | Ex: "gate_timeout_routine", "lovable_scope_drift_tolerance", "auto_approve_routine_ci" |
| rule_value | JSONB NOT NULL | |
| description | TEXT | |
| active | BOOLEAN DEFAULT true | |
| created_at / updated_at | TIMESTAMPTZ | |

**Regras seed v2.1:**

| rule_key | rule_value | Descrição |
|----------|-----------|-----------|
| `gate_timeout_routine` | `{"hours": 4}` | R2: Timeout para tasks rotina |
| `gate_timeout_complex` | `{"hours": 24}` | R2: Timeout para tasks complexas |
| `gate_timeout_critical` | `{"hours": 72}` | R2: Timeout para tasks críticas |
| `lovable_scope_drift_tolerance` | `{"max_extra_files": 0, "action": "block_gate2"}` | R1: Zero tolerância a ficheiros extra |
| `auto_approve_routine_ci` | `{"enabled": false, "gates": ["code_review", "verify"]}` | Gates auto-aprovados se CI ✅ (desactivado por defeito) |

---

## RLS Policies (v2.1 — R3)

O Dashboard usa **anon key**. A `service_role_key` do Control Plane fica **exclusivamente** nas Edge Functions (Vault secrets), nunca exposta ao frontend.

```sql
-- Leitura: anon pode ver tudo
CREATE POLICY "anon_read_projects" ON orch_projects FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_tasks" ON orch_tasks FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_plans" ON orch_plans FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_verdicts" ON orch_verdicts FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_migrations" ON orch_migrations FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_audit" ON orch_audit_log FOR SELECT TO anon USING (true);
CREATE POLICY "anon_read_rules" ON orch_rules FOR SELECT TO anon USING (true);

-- Escrita: anon só pode inserir verdicts (gates do Luis via dashboard)
CREATE POLICY "anon_insert_verdicts" ON orch_verdicts FOR INSERT TO anon
  WITH CHECK (reviewer = 'luis');

-- Zero UPDATE/DELETE via anon em qualquer tabela orch_*
-- (sem policy = negado por defeito com RLS enabled)
```

**Nota:** Edge Functions usam `service_role` para todas as escritas (state transitions, audit log, plans, migrations). O frontend é read-only excepto para verdicts do Luis.

---

## Protocolo de Comunicação

### Com Lovable (ficheiros — único canal possível)

**Protocolo de lock v2.1 (L3):**
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

### Com Perplexity (API REST) — activo desde Fase 1 (L2)
```
POST api.perplexity.ai/chat/completions
  model: "sonar-reasoning-pro"
  system: "You are a code reviewer for a Supabase+React project.
           Return JSON: {score: 1-10, findings: [{severity, description, recommendation}], approved: bool}"
  user: [conteúdo do plano/diff/SQL + contexto do projecto]
```

A chamada acontece durante a transição `PLANNING → PLAN_REVIEW`. O resultado é gravado em `orch_verdicts` com `reviewer: 'perplexity'` **antes** de o Luis ver a notificação do Gate 1. Assim, o Luis vê o score AI + findings ao decidir.

### Dashboard ← Control Plane (Supabase Realtime)
Subscriptions filtradas por `project_id` nas tabelas `orch_tasks`, `orch_verdicts`, `orch_audit_log`.

---

## Scope Drift Detection (v2.1 — R1)

### Fluxo no Gate 2 (CODE_REVIEW)

1. Claude Code (ou GitHub Action) analisa o PR/diff do Lovable
2. Extrai lista de ficheiros alterados → grava em `orch_tasks.files_affected_actual`
3. Compara com `orch_tasks.files_affected_planned` (copiado do plano)
4. Se existem ficheiros extra (não planeados):
   - Verifica regra `lovable_scope_drift_tolerance` em `orch_rules`
   - Se `max_extra_files: 0` e há extras → **bloqueia Gate 2**
   - Grava scope diff em `orch_verdicts.scope_diff`:
     ```json
     {
       "planned": ["src/foo.ts", "src/bar.ts"],
       "actual": ["src/foo.ts", "src/bar.ts", "src/baz.ts", "src/qux.ts"],
       "extra": ["src/baz.ts", "src/qux.ts"],
       "missing": []
     }
     ```
   - Notifica Luis via Telegram: "Lovable alterou 2 ficheiros fora do scope"
5. Luis decide: aprovar mesmo assim, ou rejeitar e pedir nova execução

### Dashboard — Scope Diff View

No task detail, quando `files_affected_actual` difere de `files_affected_planned`:
- Lista side-by-side: planeado vs actual
- Ficheiros extra marcados a 🔴
- Ficheiros em falta marcados a 🟡
- Botão "Override — aprovar com scope expandido" (grava override no audit log)

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

## Componentes a Construir

### Fase 1 — MVP (~2-3 weekends)

| # | Componente | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | `orch_*` schema | SQL migration | **7 tabelas** (inclui `orch_projects`) + **RLS policies (R3)** + triggers audit |
| 2 | Seed de projectos + regras | SQL | Insert get2event, capi, hr + regras de timeout e scope drift |
| 3 | `.orchestra/state.json` | Ficheiro repo | `{project_id, task_id, status, plan_version, branch}` |
| 4 | `orchestrate` Edge Function | Deno | State machine **project-aware** com scope drift check (R1) e timeout logic (R2) |
| 5 | `review-perplexity` Edge Function | Deno | **v2.1 L2:** Chamada API Perplexity durante PLANNING → PLAN_REVIEW |
| 6 | `notify-telegram` Edge Function | Deno | Push + inline buttons + **priority badge + wait time (L1)** |
| 7 | `github-webhook` Edge Function | Deno | CI status **multi-repo**: identifica projecto pelo repo URL |
| 8 | `gate-timeout-check` Edge Function | Deno | **v2.1 R2:** Scheduled (pg_cron 15min), auto-block tasks expiradas |
| 9 | `orchestra.yml` GitHub Action | YAML | Template reutilizável por repo |
| 10 | Dashboard v0 | React SPA (Lovable) | **4 páginas**: project selector, task list (sorted priority + wait — R2), task detail (scope diff — R1), audit log |

### Fase 2 — Automação (~2-3 weekends)

| # | Componente | Descrição |
|---|-----------|-----------|
| 11 | `execute-migration` Edge Function | SQL gated — **só activa se `supabase_mode = 'external'`** |
| 12 | `ingest-verdict` Edge Function | Telegram callback → orch_verdicts |
| 13 | `classify` Edge Function | Auto-classificação via Claude API |
| 14 | Auto-commit chat.md | `orchestrate` commita INBOX via GitHub API (com protocolo de lock L3) |

### Fase 3 — Polish (ongoing)

| # | Componente | Descrição |
|---|-----------|-----------|
| 15 | Schema diff view | Dashboard mostra SQL planeado vs schema real (detecta drift Lovable) |
| 16 | Manual migration assistant | Para projectos internos: gera SQL formatado + gate queries + checklist |
| 17 | n8n integration | Visual workflow builder (opcional) |
| 18 | PWA/mobile | Dashboard installable |

---

## Automatizado vs Manual (por modo Supabase)

### Tasks sem SQL (igual para ambos os modos)

| Step | Auto? | Quem? |
|------|-------|-------|
| Task creation | Manual | Luis (dashboard/Telegram) |
| Classification | Manual (Fase 1) / Auto (Fase 2) | Luis / classify() |
| Planning | Auto | Claude Code gera |
| **Perplexity AI review** | **Auto (v2.1 L2)** | **review-perplexity Edge Function** |
| **Plan review** | **GATE 1** | **Luis aprova (vê score AI + findings)** |
| Code writing | Auto | Claude Code commita |
| Lovable INBOX delivery | Manual (Fase 1) / Auto (Fase 2) | Luis / auto-commit (com lock L3) |
| **Trigger Lovable** | **PERMANENTEMENTE MANUAL** | **Luis diz "executa" no Lovable** |
| CI | Auto | GitHub Actions |
| **Scope drift check** | **Auto (v2.1 R1)** | **orchestrate compara planned vs actual** |
| **Code review** | **GATE 2** | **Luis aprova (vê scope diff se drift)** |
| **Verify** | **GATE 3** | **Luis confirma** |
| **Gate timeout** | **Auto (v2.1 R2)** | **gate-timeout-check → blocked** |

### Tasks com SQL — modo INTERNO (get2event, capi, hr)

| Step | Auto? | Quem? |
|------|-------|-------|
| SQL gerado | Auto | Claude Code |
| Gate queries geradas | Auto | Claude Code |
| **SQL colado no Lovable SQL Editor** | **PERMANENTEMENTE MANUAL** | **Luis** |
| **Gate queries executadas** | **MANUAL** | **Luis corre no SQL Editor** |
| **Confirmação** | **GATE 3** | **Luis confirma resultado** |

### Tasks com SQL — modo EXTERNO (projectos futuros)

| Step | Auto? | Quem? |
|------|-------|-------|
| SQL gerado | Auto | Claude Code |
| Pre-gate queries | Auto | execute-migration |
| SQL execution | Auto | execute-migration (transacção) |
| Post-gate queries | Auto | execute-migration |
| **Verify** | **GATE 3** | **Luis confirma** |

### Bridges irredutíveis (qualquer modo)
1. **Trigger Lovable** — Lovable não tem API
2. **Aplicar SQL em Supabase interno** — Lovable não expõe o seu Supabase externamente
3. **Gates humanos** — by design (Luis é o gate final)

---

## Tech Stack

| Camada | Escolha | Justificação |
|--------|---------|-------------|
| Control Plane DB | Supabase pessoal (wqidcmapowfudwsazblj) | Já provisionado, MCP configurado, free tier |
| Orchestration | Edge Functions (Deno) | Sem infra extra, já deployed noutros contextos |
| CI | GitHub Actions | Já existe em todos os repos |
| Notificações | Telegram Bot API | Free, instant, inline keyboards |
| Dashboard | Lovable (React+Vite+shadcn) no Control Plane Supabase | Build rápido, **anon key + RLS (R3)** |
| Com. com Lovable | `lovable/chat.md` por projecto | Único canal possível, **com protocolo de lock (L3)** |
| Estado no repo | `.orchestra/state.json` por projecto | Legível por todos os agents |
| Cross-project SQL | service_role key (só modo externo) | Vault secrets do Supabase |
| AI Review | Perplexity `sonar-reasoning-pro` | **Activo desde Fase 1 (L2)** |
| NÃO n8n | Adiado Fase 3 | Edge Functions suficientes |

### Porquê N+1 Supabase?
- **Projectos com Supabase interno** (get2event, capi, hr) — Lovable gere, sem acesso externo, pode renomear colunas
- **Projectos com Supabase externo** (futuros) — Luis gere, Control Plane conecta via service_role key
- **Control Plane** (wqidcmapowfudwsazblj) — Isolado, estável, nunca tocado pelo Lovable

---

## Verificação (como testar o MVP) — actualizado v2.1 (L4)

### Caminho feliz (happy path)
1. Registar 3 projectos (get2event, capi, hr) em `orch_projects` com `supabase_mode: 'internal'`
2. Criar 1 task no get2event via dashboard → confirmar FK project_id + priority correctos
3. Claude Code gera plano → `orch_plans` versão 1 com `sql_draft` preenchido
4. **Perplexity review automático (L2)** → `orch_verdicts` com `reviewer: 'perplexity'`, score e findings
5. Telegram recebe notificação com nome do projecto + **priority badge (L1)** → "Approve" → `orch_verdicts`
6. Dashboard mostra migration com `execution_mode: 'manual'` + SQL copiável
7. Luis cola SQL no Lovable, confirma → estado transita para VERIFY → DONE
8. Audit log completo em `orch_audit_log`

### Caminho de rejeição (v2.1 L4)
9. Criar task → plano gerado → **clicar "Request Changes"** no Gate 1
10. Confirmar que task volta a `planning` com `blocked_reason: 'changes_requested'`
11. Audit log regista transição `plan_review → planning`

### Caminho de timeout (v2.1 L4)
12. Criar task rotina → deixar em `plan_review` sem aprovar
13. Após 4h, `gate-timeout-check` transita para `blocked` com `blocked_reason: 'gate_timeout'`
14. Telegram envia notificação de timeout
15. Audit log regista `gate_timeout` com detalhes

### Scope drift (v2.1 L4)
16. Simular task onde `files_affected_actual` tem ficheiros extra vs `files_affected_planned`
17. Gate 2 bloqueia automaticamente (regra `lovable_scope_drift_tolerance`)
18. Dashboard mostra scope diff com ficheiros extra a 🔴
19. Luis override → aprovação com scope expandido gravada no audit log

### Multi-projecto
20. Repetir happy path com 2º projecto (capi) para confirmar isolamento

**Critério de sucesso:** Tasks em 2 projectos diferentes percorrem ciclos felizes e de falha simultaneamente, com audit trails separados e dashboard a mostrar ambos correctamente.

---

## Próximos Passos

1. ⏳ Luis revê arquitectura v2.1
2. Criar repo GitHub `MultiAgentAI` ✅ (https://github.com/caparicano/MultiAgenteAI)
3. Aplicar migration `orch_*` (7 tabelas + RLS) no Supabase pessoal (via MCP)
4. Seed: insert get2event, capi, hr + regras de timeout/scope
5. Criar `orchestrate` Edge Function (state machine project-aware + scope drift + timeout)
6. Criar `review-perplexity` Edge Function (AI review automático)
7. Criar `notify-telegram` Edge Function (com priority + wait time)
8. Criar `gate-timeout-check` Edge Function (scheduled)
9. Configurar Telegram Bot + secrets
10. Dashboard v0 com project selector + scope diff view (Lovable no Control Plane)
11. Teste end-to-end: happy path + rejeição + timeout + scope drift
12. Teste multi-projecto com task em capi
