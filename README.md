# MultiAgente AI

Control plane that orchestrates collaborative workflows between AI agents (Claude Code, Lovable, Perplexity) across multiple projects.

## Architecture

**GitHub-first** — uses GitHub Issues as tasks, Labels as state machine, Actions as orchestration, Projects as dashboard.

```
Luis (gates) ← GitHub Issues
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
| `/plan-ready` | Signal plan revised, trigger Perplexity review |
| `/migration-done` | Confirm SQL applied |
| `/verify-ok` | Final verification |
| `/block [reason]` | Block task |
| `/cancel` | Cancel task |

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `orchestrate.yml` | Issue events + comments | State machine transitions |
| `perplexity-review.yml` | `repository_dispatch` | AI review via Perplexity API |
| `gate-timeout.yml` | Cron `*/15 * * * *` | Auto-block expired gates |
| `chatmd-conflict.yml` | Push to `**/chat.md` | Detect lock protocol violations |
| `ci-scope-drift.yml` | Pull requests | Compare planned vs actual files |
| `prepare-dispatch.yml` | `/dispatch-review*` | Perplexity pre-flight (Phase 3) |

## Projects

| Project | Supabase | Mode |
|---------|----------|------|
| get2event | Internal (Lovable) | SQL always manual |
| capi | Internal (Lovable) | SQL always manual |
| hr | Internal (Lovable) | SQL always manual |

## Setup

1. Add GitHub Secret: `PERPLEXITY_API_KEY`
2. Fill in `lovable_project_id` in `.orchestra/projects/*.json`
3. Create first task via GitHub Issue template

## Docs

- [Architecture v3 (final)](docs/architecture/ARCHITECTURE-v3-FINAL.md)
- [Architecture history](docs/architecture/) (v2.0 → v2.2)
- [Perplexity reviews](docs/reviews/) (3 rounds of independent audit)
