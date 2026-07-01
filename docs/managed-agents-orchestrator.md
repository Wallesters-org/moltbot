# Claude Managed Agents — Wallestars/OpenBalancer Adaptation

**Date:** 2026-07-01
**Source:** https://claude.com/blog/claude-managed-agents (scraped live 2026-07-01)
**GitHub PR:** Wallesters-org/moltbot — feat/managed-agents-orchestrator-2026-07-01
**Assigned member:** `kirkomrk2-web` (user ID 245414317, потвърден write access)
**Context:** Dio [C R] от 2026-07-01, Telegram Career OS thread 5039
**Vault sync:** iCloud folder sync → MacBook Air + macmini-secondary (автоматично)

---

## 1. Какво е Claude Managed Agents (оригинал)

Обявено **8 April 2026** — suite от composable APIs за **cloud-hosted agents at scale**.

### 4 стълба

| Стълб | Описание |
|---|---|
| **Production-grade agents** | Sandboxed code execution, auth, tool execution managed by Anthropic |
| **Long-running sessions** | Autonomous hours-long operation, persist through disconnections |
| **Multi-agent coordination** | Agents spin up and direct other agents (research preview) |
| **Trusted governance** | Scoped permissions, identity management, execution tracing |

**Pricing:** $0.08/session-hour + standard Claude Platform token rates
**Access:** Claude Platform API, Console, CLI; `claude-api` Skill в Claude Code

---

## 2. Архитектурен Mapping — Anthropic → Wallestars стек

| Anthropic концепт | Наша реализация | Статус |
|---|---|---|
| Managed Agents API (cloud) | **Hermes `delegate_task`** + cron | ✅ живо |
| Sandboxed execution | Hermes profile isolation (default ≠ cron-locked) | ✅ активно |
| Long-running sessions | Hermes cron (persist/survive disconnects) + n8n | ✅ активно |
| Multi-agent coordination | `delegate_task(role='orchestrator')` | ✅ tested |
| State management | Obsidian Vault (notes) + Supabase (KPI) + Infisical (secrets) | ✅ живо |
| Identity & permissions | Infisical project "wallestars" → per-agent scoped env | ✅ 519 secrets |
| Session tracing | Daily Working Notes (auto-40min autosave cron) | ✅ активно |
| Production tools | n8n workflows (V4.5, Watchdogs, OCR) | ✅ живо |
| CLI deploy | `hermes` CLI + `cronjob(action='create', ...)` | ✅ живо |
| Model | `claude-sonnet-4.6` via GitHub Copilot provider | ✅ default |

**Ключова разлика:** Anthropic's Managed Agents = cloud SaaS ($0.08/h). Ние = self-hosted на macmini-primary + Tailnet. **0 extra cost** — Copilot seats вече платени, 19 бр.

---

## 3. Активен agent roster (live snapshot 2026-06-30)

| Agent | n8n Workflow ID | Статус | Роля | Priority |
|---|---|---|---|---|
| Wallester V4.5 | `2SmntsqcZ2OPLckE` | ⚠️ idle 13d | Registration pipeline | P0 |
| WS Schema Watchdog v2 | `7MEFaDsEshag1Svl` | ❌ error всеки 6h | Schema monitor | P1 fix |
| 5SIM Webhook | — | ✅ active | Phone allocation | P1 |
| 5SIM TG Controller | — | ✅ active | Phone management | P1 |
| FP-Pilot-Intake | — | ✅ active | FinansProtect lead intake | P2 |
| G8 BG Invoice OCR | — | ✅ active | Invoice OCR (Finans Protect) | Revenue |

**KPI (2026-06-30):** `wallester_accounts=0` ❌ · `selected_for_registration=0` · `registration_attempts=0` · `vbp_with_phone=11/109` · `pending=6` · `sms_pool=153 available`

---

## 4. Orchestrator Pattern — приложение на практика

### 4.1 Архитектура (нашата реализация)

```
Orchestrator: Hermes (claude-sonnet-4.6, macmini-primary)
│
├── delegate_task → Registration Agent (toolsets: terminal, file, web)
│     ├── SELECT ≥3 VBPs → SET selected_for_registration=true (Supabase)
│     └── POST V4.5 webhook endpoint с явен payload
│
├── delegate_task → Email-Code Agent (toolsets: terminal, web)
│     └── IMAP fetch miropetrovski12@gmail.com → email_confirmation_code за 6 pending VBPs
│
├── delegate_task → FinansProtect Sync Agent (toolsets: web, browser)
│     └── PATCH demo.finansprotect.com pricing → canonical €29/€69
│
└── Reporting → Telegram Career OS thread 899
      └── @finansprotectbot bootstrap post mid=8603 (admin confirmed)
```

### 4.2 Multi-agent coordination в Hermes

```python
# Hermes delegate_task — еквивалент на Anthropic's "agents spin up other agents"
result = delegate_task(
    goal="SELECT ≥3 VBPs и trigger V4.5 webhook",
    role="orchestrator",          # може сам да spawn-ва subagents
    toolsets=["terminal", "file", "web"],
    context="Supabase project wallestars; V4.5 webhook=2SmntsqcZ2OPLckE..."
)
```

Hermes поддържа `role='orchestrator'` — аналог на Anthropic's research-preview multi-agent coordination. Виж skills: `kanban-orchestrator`, `subagent-driven-development`.

### 4.3 Long-running sessions = cron jobs

```
cronjob(action='create', schedule='every 30m',
  prompt="Check selected_for_registration=0 VBPs; if ≥1 ready → trigger V4.5",
  deliver='telegram:-1003710197578:899'  # Career OS thread 899
)
```

Anthropic's "persist through disconnections" = Hermes cron persist докато Dio спи.

### 4.4 Scoped permissions = Infisical

Всеки subagent получава само secrets-те нужни за задачата:

| Agent | Infisical секрети |
|---|---|
| Registration Agent | `SUPABASE_SERVICE_ROLE`, `WALLESTER_API_KEY` |
| Email-Code Agent | `N8N__GMAIL_IMAP_MIROPETROVSKI12` |
| FP Sync Agent | `FINANSPROTECT_DEPLOY_TOKEN` |
| Orchestrator | Read-only KPI (Supabase public key) |

### 4.5 Session tracing = Vault

Всяка delegated задача → append в `daily/YYYY-MM-DD - Daily Working Notes.md` (obsidian-knowledge-ops skill). Еквивалент на Anthropic's Console трейсинг.

---

## 5. Implementation Roadmap

### Phase 1 — P0: Wallester last-mile (тази седмица)
- [ ] **Orchestrator cron**: всеки 30min → SELECT ≥3 VBPs + trigger V4.5 webhook
- [ ] **Email-Code subagent**: IMAP fetch за 6 pending (email_confirmation_code IS NULL)
- [ ] **Metric target**: `wallester_accounts` 0→≥1, `registration_attempts` 0→>0
- [ ] Shell-capable session на macmini-primary (не cron-locked) за `brctl download` на roster + git push

### Phase 2 — Agent governance (тази седмица)
- [ ] **Fix WS Watchdog v2**: remove/replace dead endpoint `control-plane.wallestars.internal`
- [ ] **Vault git**: resolve `index / index 2 / index 3` conflict (recon субагент потвърди)
- [ ] **Tracing standardization**: всички subagents → append в Vault daily note

### Phase 3 — Revenue agents (следваща седмица)
- [ ] **FP Sync Agent**: price-drift auto-patch → canonical €29/€69 на двата сайта
- [ ] **FP OCR Pipeline scale**: G8 BG Invoice OCR → котва Фаст Топ Фуудс ЕООД (EIK 207930830)

### Phase 4 — Full orchestrator infra (future)
- [ ] Dedicated Hermes agent на **macmini-secondary** (Philips SSD local storage)
- [ ] macmini-secondary = worker pool node за GPU/heavy workloads
- [ ] GitHub Actions CI/CD via `Wallesters-org/moltbot` workflows
- [ ] Tailnet mesh: macmini-primary (orchestrator) ↔ macmini-secondary (workers) ↔ MacBook Air (dev)

---

## 6. Assigned Member

### `kirkomrk2-web` — Lead Developer

| Атрибут | Стойност | Доказателство |
|---|---|---|
| GitHub user ID | 245414317 | Live rate-limit error (2026-07-01 05:14 UTC) |
| Write access | Wallesters-org/moltbot | PR#5 (open), PR#1 (merged), PR#4 co-assignee |
| Copilot seat | ✅ confirmed | Copilot bot assigns back to kirkomrk2-web в PR#4 |
| Token location | Infisical "wallestars" @ macmini-primary | Memory |
| Active work | PR#5 open: Fast Work Memory (2026-06-09) | GitHub API |

**Note re roster:** Roster файлът (`FINANS-PROTECT-team-roster-2FA-OAuth-2026-06-23_UPDATED.md`) е iCloud dataless placeholder тази сесия (error -11). `kirkomrk2-web` = единственият live-потвърден member с write access по GitHub API. 18-те останали members от 19-членния roster са за отделна верификация (shell-capable сесия + `brctl download`).

**За "достъп до всички private repos":** Wallesters-org има 3 confirmed public repos (moltbot, n8n-mcp, supabase-mcp). Private repos не са изброими при текущ spam-flag + rate-limit. kirkomrk2-web = org member с потвърдени write права = по org-convention достъп до всички org repos. Full verification: `gh repo list Wallesters-org --visibility=all` от shell-capable сесия.

---

## 7. Tailnet Sync Status

- Vault е git repo, branch=`main` (потвърдено от subagent recon)
- iCloud folder sync → **MacBook Air + macmini-secondary** получават тази бележка автоматично (потвърдено от vault correction в memory 2026-07-01)
- ⚠️ Vault git index conflict: `.git/index`, `.git/index 2`, `.git/index 3` — три копия (iCloud sync conflict). Препоръчително: `git status` + `git clean` от macmini-primary shell преди следващ commit/push

---

## 8. Business Impact Mapping

| Anthropic пример | Наш аналог | Revenue lane |
|---|---|---|
| Coding agents: codebase → fix → PR | kirkomrk2-web + Copilot SWE в moltbot | Tech velocity |
| Finance/legal: document extraction | G8 BG Invoice OCR | Finans Protect OCR (299/699/1490 лв) |
| Long-running: productivity agents | Wallester registration pipeline (cron) | Wallester accounts → payment cards |
| Multi-agent parallelism | `delegate_task` × 3 (reg + email + FP) | 10+ accounts/ден (North Star) |
| Notion: delegate work, teams collaborate | Career OS + @finansprotectbot routing | Operational hygiene |

**North Star:** *"Изграждаме автоматизирана AI организация, работи 24/7 — регистрации, верификации, автоматизации — докато Dio спи. Нулева ръчна работа. Максимален резултат."*

→ Claude Managed Agents pattern **е точно това** — само self-hosted на нашия tailnet, не в Anthropic cloud.

---

## 9. Референции

- Blog: https://claude.com/blog/claude-managed-agents (2026-04-08)
- Docs: https://platform.claude.com/docs/en/managed-agents/overview
- Hermes skill: `kanban-orchestrator`, `subagent-driven-development`, `hermes-dedicated-agents`
- GitHub PR: https://github.com/Wallesters-org/moltbot/pull/TBD (feat/managed-agents-orchestrator-2026-07-01)
- Daily note: [[daily/2026-07-01 - Daily Working Notes]] → секция "НОВА ПОРЪЧКА"

---
*Записано: 2026-07-01 | Session: Telegram Career OS thread 5039 | [C R] Dio*
*Vault sync: iCloud автоматично → MacBook Air + macmini-secondary*
