# MultiAgente AI

Control plane that orchestrates collaborative workflows between AI agents (Claude Code, Lovable, Perplexity) across multiple projects.

## Architecture

**GitHub-first** — uses GitHub Issues as tasks, Labels as state machine, Actions as orchestration, Projects as dashboard.

```
Luis (gates) ← Telegram / GitHub
       ↓
   GitHub Issues + Actions + Projects
       ↓              ↓              ↓
  Claude Code      Lovable      Perplexity API
  (planner)     (implementer)    (reviewer)
```

## State Machine

```
intake → planning → plan-review → execution → code-review → migration → verify → done
                    (GATE 1)                   (GATE 2)                  (GATE 3)
```

## Slash Commands

| Command | Effect |
|---------|--------|
| `/approve` | Approve current gate |
| `/changes [text]` | Request changes |
| `/migration-done` | Confirm SQL applied |
| `/verify-ok` | Final verification |
| `/block [reason]` | Block task |
| `/cancel` | Cancel task |

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `orchestrate.yml` | Issue events + comments | State machine transitions |
| `review-perplexity.yml` | `status:planning` label | AI review via Perplexity API |
| `notify-telegram.yml` | Called by other workflows | Telegram notifications (3x retry) |
| `gate-timeout.yml` | Cron `*/15 * * * *` | Auto-block expired gates |
| `chatmd-conflict.yml` | Push to `**/chat.md` | Detect lock protocol violations |
| `ci-scope-drift.yml` | Pull requests | Compare planned vs actual files |
| `dispatch-prepare.yml` | `/dispatch-review*` | Perplexity pre-flight (Phase 3) |

## Projects

| Project | Supabase | Mode |
|---------|----------|------|
| get2event | Internal (Lovable) | SQL always manual |
| capi | Internal (Lovable) | SQL always manual |
| hr | Internal (Lovable) | SQL always manual |

## Setup

1. Create Telegram bot via @BotFather
2. Add GitHub Secrets: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`, `PERPLEXITY_API_KEY`
3. Fill in `lovable_project_id` in `.orchestra/projects/*.json`
4. Create first task via GitHub Issue template

## Docs

- [Architecture v3 (final)](docs/architecture/ARCHITECTURE-v3-FINAL.md)
- [Architecture history](docs/architecture/) (v2.0 → v2.2)
- [Perplexity reviews](docs/reviews/) (3 rounds of independent audit)
