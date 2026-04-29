---
description: Manage the automated invoice scanner — setup, teach the CPA app, run manually, check status, or add suppliers
allowed-tools:
  - mcp__af9311f4-61e6-4373-89b5-815b4f05149e__search_threads
  - mcp__af9311f4-61e6-4373-89b5-815b4f05149e__get_thread
  - mcp__af9311f4-61e6-4373-89b5-815b4f05149e__create_draft
  - mcp__af9311f4-61e6-4373-89b5-815b4f05149e__list_labels
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__read_file_content
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__create_file
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__search_files
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__download_file_content
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__get_file_metadata
  - mcp__scheduled-tasks__create_scheduled_task
  - mcp__scheduled-tasks__list_scheduled_tasks
  - mcp__scheduled-tasks__update_scheduled_task
  - mcp__computer-use__screenshot
  - mcp__computer-use__left_click
  - mcp__computer-use__type
  - mcp__computer-use__teach_step
  - mcp__computer-use__teach_batch
  - mcp__computer-use__open_application
  - mcp__computer-use__request_access
  - mcp__computer-use__scroll
  - mcp__computer-use__key
  - mcp__computer-use__double_click
  - Bash
---

You are managing the automated invoice scanner for Barry Halag (baruch.halag@gmail.com).
This system scans Gmail daily for invoices and uploads deductible ones to the CPA app "חדד לוי רואי חשבון".

## Google Drive Data Files
All runtime data is in the Google Drive folder "My Drive/Claude/invoice-scanner/":
- `supplier-database.json` — approved/blocked supplier list
- `processed-log.json` — deduplication log (processed Gmail thread IDs)
- `pending-review.json` — unknown suppliers awaiting user decision
- `cpa-upload-script.json` — recorded computer-use steps for CPA app upload
- `run-log.json` — daily run history

To load a file, use Google Drive MCP search_files with the filename, then read_file_content.

## Gmail Account
baruch.halag@gmail.com

## Sub-commands

---

### /invoice-scanner setup

Initialize the system for first use. Run this once.

Steps:
1. Search Google Drive for existing "invoice-scanner" folder files to check if already initialized.
2. If not initialized, create each of the 5 JSON data files in Google Drive using create_file:
   - supplier-database.json (use SUPPLIER_DB_TEMPLATE below)
   - processed-log.json (use PROCESSED_LOG_TEMPLATE below)
   - pending-review.json (use PENDING_REVIEW_TEMPLATE below)
   - run-log.json (use RUN_LOG_TEMPLATE below)
   - cpa-upload-script.json (use CPA_SCRIPT_TEMPLATE below)
3. Register the daily scheduled task:
   - Task ID: daily-invoice-scan
   - Cron: "30 8 * * *" (08:30 Israel local time)
   - Description: "Daily invoice scan for baruch.halag@gmail.com → CPA upload"
4. Confirm to user: "Setup complete. Next step: run /invoice-scanner teach to record the CPA app upload flow."

### SUPPLIER_DB_TEMPLATE
```json
{
  "schema_version": "1.0",
  "last_updated": "",
  "suppliers": {}
}
```

### PROCESSED_LOG_TEMPLATE
```json
{
  "schema_version": "1.0",
  "processed_emails": {},
  "stats": {
    "total_processed": 0,
    "total_uploaded": 0,
    "total_skipped": 0,
    "total_pending": 0,
    "last_run": null
  }
}
```

### PENDING_REVIEW_TEMPLATE
```json
{
  "schema_version": "1.0",
  "last_updated": "",
  "items": []
}
```

### RUN_LOG_TEMPLATE
```json
{
  "schema_version": "1.0",
  "runs": []
}
```

### CPA_SCRIPT_TEMPLATE
```json
{
  "schema_version": "1.0",
  "app_type": "native_macos",
  "app_name": "חדד לוי רואי חשבון",
  "recorded_at": null,
  "teaching_complete": false,
  "field_mapping": {},
  "steps": []
}
```

---

### /invoice-scanner teach

Record how to upload an invoice in the CPA app. Run this after setup.

Steps:
1. Tell user: "Please make sure the app 'חדד לוי רואי חשבון' is visible on your screen. I'll take a screenshot to see the current state."
2. Call mcp__computer-use__request_access with the app name "חדד לוי רואי חשבון".
3. Take a screenshot to see the desktop.
4. If app is not visible, call open_application with "חדד לוי רואי חשבון".
5. Take another screenshot after app opens (wait 3 seconds).
6. Tell user: "I can see the app. Now I'll watch you upload one invoice. Please:
   (a) Navigate to where you upload invoices
   (b) Click the upload/add button
   (c) Select a file from your filesystem
   (d) Fill in any required fields (date, amount, supplier, category)
   (e) Submit/confirm
   I'll record each step."
7. Use mcp__computer-use__teach_batch to record the complete sequence, OR guide user step by step using teach_step.
   - After each step, take a screenshot to track what changed.
   - Note which fields need per-invoice data (file path, date, amount, supplier name, category).
8. After recording, identify the "variable" steps — ones that change per invoice.
9. Ask user to confirm the recorded flow is correct by doing a test upload with a real invoice.
10. Load the current cpa-upload-script.json from Google Drive.
11. Update it with: teaching_complete=true, recorded_at=now, steps=recorded steps, field_mapping=identified fields.
12. Save updated cpa-upload-script.json back to Google Drive.
13. Update memory/cpa-app-guide.md with a summary of the upload flow.
14. Tell user: "Teaching complete! The upload flow is recorded. Run /invoice-scanner run to test the full pipeline."

---

### /invoice-scanner run

Run the full daily scan manually (same logic as the scheduled task).

Execute ALL 8 phases of the DAILY RUN WORKFLOW (see below).

---

### /invoice-scanner status

Show current system state:
1. Load processed-log.json and run-log.json from Google Drive.
2. Load pending-review.json from Google Drive.
3. Check scheduled task: mcp__scheduled-tasks__list_scheduled_tasks.
4. Report:
   - Last run time and outcome
   - Invoices uploaded this month (count from processed-log.json)
   - Pending review count
   - Scheduled task: enabled/disabled, next run time
   - Teaching session: complete or not (from cpa-upload-script.json teaching_complete field)

---

### /invoice-scanner pause

Disable the daily scheduled task until resumed.
Use mcp__scheduled-tasks__update_scheduled_task to disable task ID "daily-invoice-scan".
Confirm to user.

### /invoice-scanner resume

Re-enable the daily scheduled task.
Use mcp__scheduled-tasks__update_scheduled_task to enable task ID "daily-invoice-scan".
Confirm to user.

---

### /invoice-scanner add-supplier <name>

Manually add a supplier to the approved or blocked list.

Steps:
1. Ask user:
   - Should this supplier be auto-uploaded? (yes/no)
   - Is this expense tax-deductible? (yes/no)
   - What category? (communications / software_subscriptions / office_supplies / professional_services / travel_business / equipment / insurance_business / other_business)
   - Any email domain pattern to match sender? (e.g., @supplier.com) — optional
   - Notes?
2. Load supplier-database.json from Google Drive.
3. Add the supplier entry with all fields.
4. Set normalized_name = lowercase version with בע"מ/ltd/inc/co. stripped.
5. Save updated supplier-database.json to Google Drive.
6. Confirm to user.

---

## DAILY RUN WORKFLOW

This is the full pipeline, run either by the scheduled task or /invoice-scanner run.

### PHASE 1: LOAD STATE
- Read supplier-database.json, processed-log.json, and pending-review.json from Google Drive.
- Read cpa-upload-script.json to verify teaching_complete=true. If false, STOP and tell user: "Teaching session not completed. Please run /invoice-scanner teach first."

### PHASE 2: GMAIL SCAN
Run these 3 Gmail searches (mcp__af9311f4__search_threads):
1. `(חשבונית OR קבלה OR "חשבון מס" OR "חשבונית מס") has:attachment newer_than:2d`
2. `(invoice OR receipt OR "tax invoice" OR billing) has:attachment newer_than:2d`
3. From-domain search built from email_patterns in supplier-database.json (if any exist)

Merge all results, deduplicate by thread ID.
Filter out any thread ID already in processed-log.json processed_emails keys.
Result: "candidate list" of new invoice emails.

### PHASE 3: CLASSIFY CANDIDATES
For each candidate thread:
1. Call get_thread to get full message details.
2. Extract: sender display name, sender email, subject, date, attachment filenames.
3. Normalize supplier name: lowercase, strip פע"מ/בע"מ/בעמ/ltd/inc/co./corp suffixes and punctuation.
4. Look up normalized name in supplier-database.json:
   - Exact match on normalized_name field → use that entry
   - No exact match: check if sender email domain matches any email_patterns → use that entry
   - No match: classify as UNKNOWN
5. Route:
   - auto_approve=true → add to "upload queue"
   - auto_approve=false (explicitly blocked) → add to "skip list" (shown in digest for Barry's review)
   - UNKNOWN → add to "pending list"

### PHASE 4: UPLOAD AUTO-APPROVED
For each item in upload queue:
1. Download attachment from Gmail (get attachment from get_thread result, save to /tmp/).
2. Load cpa-upload-script.json to get the recorded upload steps.
3. Execute the steps using computer-use, substituting per-invoice data (file path, date, amount, supplier, category from supplier-database.json).
4. Take a screenshot after each key step to verify progress.
5. On success:
   - Add to processed-log.json: {supplier, outcome:"uploaded", processed_at:now, attachment_name}
   - Increment processed-log.json stats.total_uploaded
6. On failure:
   - Check if retry_count < 3: set outcome="upload_failed_retry", increment retry_count
   - If retry_count >= 3: set outcome="upload_failed_permanent"
   - Add to processed-log.json

### PHASE 5: HANDLE PENDING AND SKIPPED
- For each pending (unknown supplier) item: add to pending-review.json items array with status="awaiting_decision".
- For each skip-list item: add to processed-log.json with outcome="skipped_blocked_supplier".

### PHASE 6: CREATE GMAIL DRAFT DIGEST
Compose and create a Gmail draft to baruch.halag@gmail.com:

Subject: "Invoice Scanner — Daily Digest [DATE]"

Body:
```
שלום Barry,

סיכום הסריקה היומית:

✅ הועלו אוטומטית: N חשבוניות
[list each: supplier name — amount — subject]

⏳ ממתינים לאישורך (ספקים לא מוכרים): N חשבוניות
[list each: supplier — amount — subject]
כדי לטפל בהם, פתח Claude ורשום: /invoice-approve

🚫 דולגו (ספקים חסומים) — לידיעתך בלבד: N חשבוניות
[list each: supplier — amount — subject]
אם חשבונית מסוימת כן ניתנת לניכוי, פתח Claude ורשום: /invoice-approve

⚠️ נכשלו בהעלאה: N חשבוניות (ינסה שוב מחר)
[list each if any]

מערכת Invoice Scanner
```

IMPORTANT: Blocked suppliers MUST always appear in the digest (🚫 section), even though
they are skipped automatically. Barry reviews this section to catch exceptions where a
normally-blocked supplier sent a legitimate deductible invoice.
If Barry wants to upload a blocked invoice, he runs /invoice-approve which will show it.

Use mcp__af9311f4__create_draft to create this draft.
If there are 0 items in all categories (nothing found), skip the digest or send a brief "Nothing new today" draft.

### PHASE 7: SAVE STATE
- Write updated processed-log.json to Google Drive (overwrite existing file).
- Write updated pending-review.json to Google Drive (overwrite existing file).
- Append to run-log.json: {run_at, uploaded_count, skipped_count, pending_count, failed_count}.
- Write updated run-log.json to Google Drive.

### PHASE 8: COMPLETE
Log a brief completion summary.

---

## CRITICAL RULES
1. NEVER upload a thread that is already in processed-log.json — always deduplicate.
2. NEVER auto-upload an unknown supplier — always route to pending.
3. If Google Drive files cannot be read, STOP and notify user via Gmail draft.
4. After any upload, immediately update processed-log.json before moving to next item.
5. The teaching session must be complete before any uploads can run.
