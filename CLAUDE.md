# Claude Working Directory — Barry Halag

## Invoice Scanner System
Automated daily invoice scanning from Gmail → upload to CPA app (חדד לוי רואי חשבון).
Runs every day at 08:30 Israel time.

### Commands
- `/invoice-scanner` — setup, teach, manual run, status, add-supplier
- `/invoice-approve` — review and process pending invoices from unknown suppliers

### Runtime Data (Google Drive)
All JSON data files live in `invoice-scanner/` subfolder:
- `supplier-database.json` — approved/blocked supplier list
- `processed-log.json` — deduplication log (prevents double-uploads)
- `pending-review.json` — unknown suppliers awaiting your decision
- `cpa-upload-script.json` — recorded computer-use steps for the CPA app
- `run-log.json` — daily run history and stats

### Memory
- `memory/invoice-scanner.md` — system state, file locations, scheduled task info
- `memory/cpa-app-guide.md` — CPA app upload instructions (filled after teaching session)

### GitHub
Source code and skill files: https://github.com/barryhalag/invoice-scanner-skill

## Important Rules
- NEVER store invoice PDFs or financial data in memory files — always use Google Drive
- Runtime JSON data files are NOT committed to git (see .gitignore)
- When skill files change, push to GitHub to maintain version history
