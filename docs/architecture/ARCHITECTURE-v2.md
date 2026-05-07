# MultiAgent AI — Plano de Arquitectura Consolidado (v2)

> Síntese das visões de Claude Code, Lovable e Perplexity + clarificações do Luis.
> Plataforma **multi-projecto**, **multi-sessão**, com Supabase **configurável** por projecto.
> **Status: DRAFT — pendente revisão humana**

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
 │    orch_tasks        (FK → project, multi-sessão)                │
 │    orch_plans        (versionados, FK → task)                    │
 │    orch_verdicts     (decisões por gate)                         │
 │    orch_migrations   (SQL audit trail)                           │
 │    orch_audit_log    (append-only, imutável)                     │
 │    orch_rules        (classificação, config)                     │
 │                                                                   │
 │  Edge Functions:                                                  │
 │    orchestrate         (state machine, project-aware)             │
 │    notify-telegram     (push + inline approve/reject)             │
 │    github-webhook      (CI status, multi-repo)                   │
 │    execute-migration   (só projectos com Supabase EXTERNO)       │
 │    ingest-verdict      (Telegram callback)                       │
 │    classify            (auto-classificação)                      │
 │                                                                   │
 │  Dashboard SPA (React+Vite+shadcn):                              │
 │    Project selector → Task list → Task detail → Audit log        │
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
| **Perplexity** | Revisor #2: verdicts + risks + citações | Não |
| **Lovable** | Implementador: código + SQL no seu Supabase interno | Sim (Supabase interno). Não (Supabase externo — não tem acesso) |
| **Control Plane** | Orquestração, audit, secrets, notificações | Só Supabase externo (via service_role key) |

---

## State Machine (7 estados)

```
INTAKE ──→ PLANNING ──→ PLAN_REVIEW ──→ EXECUTION ──→ CODE_REVIEW ──→ MIGRATION ──→ VERIFY ──→ DONE
                         ▲ GATE 1       (pode loop    ▲ GATE 2       (se SQL)      ▲ GATE 3
                         │ (humano)      com Lovable)  │ (humano)                   │ (humano)
                         │                             │                            │
                         └── changes_requested ────────┘                            │
                                                                                    │
                         └──────────── blocked / cancelled ─────────────────────────┘
```

**Nota:** O estado MIGRATION comporta-se diferentemente conforme o modo Supabase:
- **Interno:** Control Plane gera SQL, notifica Luis, Luis cola no Lovable SQL Editor, Luis confirma → VERIFY
- **Externo:** Control Plane executa SQL via `execute-migration` (com 5 camadas defesa) → VERIFY

### Classificação (determina canais + automação)

| Tipo | Critério | Canal primário | Gates automáticos |
|------|----------|---------------|-------------------|
| **Rotina** | Sem SQL, single-agent | Telegram push + botão inline | Gates 2-3 podem ser auto se CI ✅ |
| **Complexa** | Envolve Lovable ou SQL | Dashboard + Telegram | Todos manuais |
| **Crítica** | RLS, auth, drop, alter sensível | Dashboard + email | Todos manuais + confirmação dupla |

---

## Data Model (7 tabelas, prefixo `orch_`)

No Supabase pessoal (wqidcmapowfudwsazblj), isolado das tabelas existentes:

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

### `orch_tasks` (FK → project)

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| **project_id** | UUID FK → orch_projects | **Qual projecto** |
| title | TEXT NOT NULL | |
| description | TEXT | |
| source | TEXT DEFAULT 'manual' | manual / telegram / github_issue |
| classification | TEXT DEFAULT 'routine' | routine / complex / critical |
| status | TEXT DEFAULT 'intake' | intake → planning → plan_review → execution → code_review → migration → verify → done / blocked / cancelled |
| target_branch | TEXT | |
| pr_number | INTEGER | |
| assigned_agents | JSONB DEFAULT '[]' | ["claude_code","lovable","perplexity"] |
| metadata | JSONB DEFAULT '{}' | |
| created_at / updated_at | TIMESTAMPTZ | |

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

### `orch_verdicts`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| task_id | UUID FK | |
| gate | TEXT NOT NULL | plan_review / code_review / verify |
| decision | TEXT NOT NULL | approved / changes_requested / rejected |
| reviewer | TEXT DEFAULT 'luis' | |
| comments | TEXT | |
| ai_score | INTEGER | Score 1-10 (se review AI) |
| ai_findings | JSONB | Findings estruturados (se review AI) |
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
| action | TEXT NOT NULL | state_transition / commit / plan_written / verdict / migration_applied / notification_sent |
| from_state | TEXT | |
| to_state | TEXT | |
| details | JSONB DEFAULT '{}' | |
| created_at | TIMESTAMPTZ | |

### `orch_rules`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| id | UUID PK | |
| project_id | UUID FK (nullable) | NULL = regra global |
| rule_key | TEXT NOT NULL | Ex: "auto_approve_routine_ci", "lovable_file_scope" |
| rule_value | JSONB NOT NULL | |
| description | TEXT | |
| active | BOOLEAN DEFAULT true | |
| created_at / updated_at | TIMESTAMPTZ | |

---

## Protocolo de Comunicação

### Com Lovable (ficheiros — único canal possível)
```
{project.lovable_chat_path}     ← INBOX/OUTBOX (configurável por projecto)
{project.orchestra_path}/state.json  ← Estado machine-readable
{project.orchestra_path}/current-plan.md ← Plano actual
```

### Com Claude Code (MCP + gh CLI)
- Supabase MCP: `execute_sql` para ler/escrever orch_tables no Control Plane
- GitHub: `gh pr/issue/api` para operações em qualquer repo registado
- HTTP: `fetch()` para Perplexity API

### Com Perplexity (API REST)
```
POST api.perplexity.ai/chat/completions
  model: "sonar-reasoning-pro"
  system: "You are a code reviewer. Return JSON: {score, findings, approved}"
  user: [conteúdo do plano/diff/SQL + contexto do projecto]
```

### Dashboard ← Control Plane (Supabase Realtime)
Subscriptions filtradas por `project_id` nas tabelas `orch_tasks`, `orch_verdicts`, `orch_audit_log`.

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
| 1 | `orch_*` schema | SQL migration | **7 tabelas** (inclui `orch_projects`) + RLS + triggers audit |
| 2 | Seed de projectos | SQL | Insert get2event, capi, hr com `supabase_mode: 'internal'` |
| 3 | `.orchestra/state.json` | Ficheiro repo | `{project_id, task_id, status, plan_version, branch}` |
| 4 | `orchestrate` Edge Function | Deno | State machine **project-aware**: recebe `{project_id, task_id, action}` |
| 5 | `notify-telegram` Edge Function | Deno | Push + inline buttons, inclui nome do projecto na mensagem |
| 6 | `github-webhook` Edge Function | Deno | CI status **multi-repo**: identifica projecto pelo repo URL |
| 7 | `orchestra.yml` GitHub Action | YAML | Template reutilizável por repo |
| 8 | Dashboard v0 | React SPA (Lovable) | **4 páginas**: project selector, task list, task detail, audit log |

### Fase 2 — Automação (~2-3 weekends)

| # | Componente | Descrição |
|---|-----------|-----------|
| 9 | `execute-migration` Edge Function | SQL gated — **só activa se `supabase_mode = 'external'`**. Ignora projectos internos. |
| 10 | `ingest-verdict` Edge Function | Telegram callback → orch_verdicts |
| 11 | `classify` Edge Function | Auto-classificação via Claude API |
| 12 | Perplexity integration | `orchestrate` chama API durante PLANNING |
| 13 | Auto-commit chat.md | `orchestrate` commita INBOX via GitHub API |

### Fase 3 — Polish (ongoing)

| # | Componente | Descrição |
|---|-----------|-----------|
| 14 | Schema diff view | Dashboard mostra SQL planeado vs schema real (detecta drift Lovable) |
| 15 | Manual migration assistant | Para projectos internos: gera SQL formatado + gate queries + checklist para colar no Lovable |
| 16 | n8n integration | Visual workflow builder (opcional) |
| 17 | PWA/mobile | Dashboard installable |

---

## Automatizado vs Manual (por modo Supabase)

### Tasks sem SQL (igual para ambos os modos)

| Step | Auto? | Quem? |
|------|-------|-------|
| Task creation | Manual | Luis (dashboard/Telegram) |
| Classification | Manual (Fase 1) / Auto (Fase 2) | Luis / classify() |
| Planning | Auto | Claude Code gera |
| **Plan review** | **GATE 1** | **Luis aprova** |
| Code writing | Auto | Claude Code commita |
| Lovable INBOX delivery | Manual (Fase 1) / Auto (Fase 2) | Luis / auto-commit |
| **Trigger Lovable** | **PERMANENTEMENTE MANUAL** | **Luis diz "executa" no Lovable** |
| CI | Auto | GitHub Actions |
| **Code review** | **GATE 2** | **Luis aprova** |
| **Verify** | **GATE 3** | **Luis confirma** |

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
| Dashboard | Lovable (React+Vite+shadcn) no Control Plane Supabase | Build rápido |
| Com. com Lovable | `lovable/chat.md` por projecto | Único canal possível |
| Estado no repo | `.orchestra/state.json` por projecto | Legível por todos os agents |
| Cross-project SQL | service_role key (só modo externo) | Vault secrets do Supabase |
| NÃO n8n | Adiado Fase 3 | Edge Functions suficientes |

### Porquê N+1 Supabase?
- **Projectos com Supabase interno** (get2event, capi, hr) — Lovable gere, sem acesso externo, pode renomear colunas
- **Projectos com Supabase externo** (futuros) — Luis gere, Control Plane conecta via service_role key
- **Control Plane** (wqidcmapowfudwsazblj) — Isolado, estável, nunca tocado pelo Lovable

---

## Verificação (como testar o MVP)

1. Registar 3 projectos (get2event, capi, hr) em `orch_projects` com `supabase_mode: 'internal'`
2. Criar 1 task no get2event via dashboard → confirmar FK project_id correcto
3. Claude Code gera plano → `orch_plans` versão 1 com `sql_draft` preenchido
4. Telegram recebe notificação com nome do projecto → "Approve" → `orch_verdicts`
5. Dashboard mostra migration com `execution_mode: 'manual'` + SQL copiável
6. Luis cola SQL no Lovable, confirma → estado transita para VERIFY → DONE
7. Audit log completo em `orch_audit_log`
8. Repetir com 2º projecto (capi) para confirmar multi-projecto

**Critério de sucesso:** Tasks em 2 projectos diferentes percorrem o ciclo completo simultaneamente, com audit trails separados e dashboard a mostrar ambos.

---

## Próximos Passos

1. ⏳ Luis revê arquitectura
2. Criar repo GitHub `MultiAgentAI` (conta pessoal)
3. Aplicar migration `orch_*` (7 tabelas) no Supabase pessoal (via MCP)
4. Seed: insert get2event, capi, hr como projectos internos
5. Criar `orchestrate` Edge Function (state machine project-aware)
6. Criar `notify-telegram` Edge Function
7. Configurar Telegram Bot + secrets
8. Dashboard v0 com project selector (Lovable no Control Plane)
9. Teste end-to-end com 1 task real em get2event
10. Teste multi-projecto com task em capi
