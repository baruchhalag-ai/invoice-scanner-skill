# Invoice Scanner — Architecture

## Overview

```
Gmail (baruch.halag@gmail.com)
  ↓ daily 08:30 Israel time (cron: 30 8 * * *)
Scheduled Claude Agent (scheduled-tasks/daily-invoice-scan/SKILL.md)
  ↓ reads
supplier-database.json (Google Drive)
  ↓ routes to
  ├─ auto_approve=true  → computer-use upload → CPA app (חדד לוי רואי חשבון)
  ├─ auto_approve=false → skipped, logged
  └─ unknown supplier   → pending-review.json + Gmail draft digest
                                    ↓
                            /invoice-approve session
                            (user reviews, makes decision)
                            updates supplier-database.json
```

## Components

### 1. Command Files (.claude/commands/)
- `invoice-scanner.md` — master command with sub-commands: setup, teach, run, status, pause, resume, add-supplier
- `invoice-approve.md` — interactive approval workflow for pending invoices

### 2. Scheduled Task (scheduled-tasks/daily-invoice-scan/SKILL.md)
Runs the 8-phase daily pipeline. Registered via `mcp__scheduled-tasks__create_scheduled_task`.

### 3. Google Drive Data (invoice-scanner/)
| File | Description |
|------|-------------|
| supplier-database.json | Supplier decisions: auto_approve, tax_deductible, category |
| processed-log.json | Thread IDs already processed — prevents duplicate uploads |
| pending-review.json | Unknown suppliers waiting for Barry's decision |
| cpa-upload-script.json | Recorded computer-use steps for the CPA macOS app |
| run-log.json | History of every daily run |

### 4. Memory Files (~/.claude/projects/.../memory/)
- `invoice-scanner.md` — system state summary (not a data store — always read Google Drive for live data)
- `cpa-app-guide.md` — human-readable summary of the CPA app upload flow
- `user-profile.md` — Barry's role and preferences

### 5. CLAUDE.md (project root)
Auto-loaded by Claude Code every session. Points to commands and memory.

## Daily Run Phases
1. Load state from Google Drive
2. Gmail scan (3 search passes: Hebrew terms, English terms, known sender domains)
3. Deduplicate against processed-log.json
4. Classify: auto-approve / skip / pending
5. Upload auto-approved via computer-use
6. Add unknowns to pending-review.json
7. Create Gmail draft digest
8. Save all state back to Google Drive

## Supplier Decision Logic
```
Extract "From:" display name
  → Normalize (lowercase, strip legal suffixes)
  → Exact match in supplier-database.normalized_name?
      YES → use that entry's auto_approve value
      NO → Check sender email domain vs email_patterns?
          YES → use that entry's auto_approve value
          NO → UNKNOWN → add to pending-review.json
```

## Scheduling
- Tool: `mcp__scheduled-tasks__create_scheduled_task`
- Cron: `30 8 * * *`
- Timezone: machine local time (Israel time — IDT UTC+3 summer, IST UTC+2 winter)
- Task ID: `daily-invoice-scan`

## CPA App Integration
- App type: native macOS app
- Interaction: `mcp__computer-use__*` tools
- Script: recorded via `teach_batch`/`teach_step` during teaching session
- Parametrized fields: file path, date, amount, supplier, category (per-invoice)
- Retry logic: up to 3 attempts per invoice, then marks as permanent failure

## Version Control
- GitHub repo: https://github.com/barryhalag/invoice-scanner-skill
- What's versioned: skill files, schemas, templates, docs
- What's NOT versioned: runtime JSON data (supplier-database, logs, etc.) — stays in Google Drive
