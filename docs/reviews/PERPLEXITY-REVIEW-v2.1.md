# MultiAgente AI — Revisao Independente #2 por Perplexity
> 7 de Maio de 2026 | Baseada no ARCHITECTURE-v2.1.md
> Todos os findings foram incorporados na v2.2

## Findings

### P1 — Dashboard sem Autenticacao
- Qualquer pessoa com URL podia ver projectos, planos e SQL drafts
- anon key le tudo sem restricao
- **Resolucao v2.2:** Supabase Auth magic link (luis.costa@get2c.pt). Zero acesso anon. RLS filtrada por authenticated.

### P2 — Perplexity API Key nao Documentada
- Gap operacional: secrets nao detalhados
- **Resolucao v2.2:** Vault secrets documentados: PERPLEXITY_API_KEY, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, SERVICE_ROLE_KEY_{project_id}

### P3 — Lock chat.md Fragil
- Conflito silencioso se Lovable sobrescrever durante INBOX_PENDING
- **Resolucao v2.2:** GitHub Action chatmd-conflict-check.yml detecta commits conflitantes e notifica Luis

### P4 — Fase 1 com 10 Componentes e Demais
- Risco de ficar a meio sem nada funcional
- **Resolucao v2.2:** Fase 1 dividida em 1A (motor: 7 componentes, CLI + Telegram) e 1B (UI + AI: 4 componentes, dashboard + Perplexity)

### P5 — pg_cron no Free Tier
- pg_cron pode nao estar disponivel no Supabase free tier
- **Resolucao v2.2:** Supabase Cron Jobs (feature nativa). Fallback: GitHub Actions scheduled (cron: '*/15 * * * *')

## O que Ficou Bem na v2.1
- Scope drift detection (R1) — correctamente desenhado
- Gate timeouts (R2) — pragmaticos
- RLS base (R3) — melhorada em v2.2 com auth
- State machine clara com 7 estados
- Audit log imutavel com trigger
