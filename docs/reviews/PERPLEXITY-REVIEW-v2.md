# MultiAgente AI — Revisao Independente por Perplexity
> 7 de Maio de 2026 | Baseada no commit a3e754f (ARCHITECTURE-v2.md)
> Todos os findings foram incorporados na v2.1

## Riscos Altos Identificados

### R1 — Lovable como Implementador sem Verificacao de Scope
- Lovable ja fez scope creep (6 ficheiros pedidos, 13+ executados) e schema drift
- Protocolo chat.md nao inclui verificacao pos-execucao
- **Resolucao v2.1:** Campo `files_affected_actual` + comparacao automatica no Gate 2 + regra `lovable_scope_drift_tolerance`

### R2 — Luis como Single Point of Failure nos Gates
- Sem timeouts, SLA ou priorizacao entre tasks
- Com multiplos projectos, Luis torna-se bottleneck
- **Resolucao v2.1:** Gate timeouts (4h/24h/72h), campo `priority`, auto-block por timeout, dashboard sorted by priority + wait time

### R3 — Dashboard no Lovable com Acesso ao Control Plane
- Risco de service_role_key exposta ao frontend
- **Resolucao v2.1:** Dashboard usa anon key + RLS. service_role exclusivamente em Edge Functions (Vault)

## Lacunas Identificadas

### L1 — Sem Gestao de Conflitos Multi-Projecto
- Telegram envia notificacoes sem contexto de prioridade
- **Resolucao v2.1:** Badge de prioridade + contagem de tasks em espera nas notificacoes

### L2 — Perplexity Ausente na Fase 1
- Revisor #2 so aparecia na Fase 2, Gate 1 era puramente manual
- **Resolucao v2.1:** review-perplexity Edge Function promovida para Fase 1. Chamada automatica durante PLANNING -> PLAN_REVIEW

### L3 — Sem Versionamento/Conflito no chat.md
- Claude Code e Lovable podem escrever simultaneamente
- **Resolucao v2.1:** Protocolo de lock: STATUS header (CLEAR / INBOX_PENDING / OUTBOX_READY)

### L4 — MVP so Testa Caminho Feliz
- Cenario de verificacao nao testa rejeicao, timeout, blocked
- **Resolucao v2.1:** 4 cenarios de teste: happy path, rejeicao, timeout, scope drift

## O que Estava Bem Resolvido
- Distincao Supabase interno vs externo (decisao arquitectural mais importante)
- State machine de 7 estados com gates humanos bem posicionados
- Audit log imutavel (trigger bloqueia UPDATE/DELETE)
- Bridges irredutiveis honestos (nao tenta automatizar o impossivel)
