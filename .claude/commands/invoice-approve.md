---
description: Review and approve or reject pending invoices from unknown suppliers, or re-open a blocked invoice for upload
allowed-tools:
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__read_file_content
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__create_file
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__search_files
  - mcp__69f0d116-75ea-46c6-82ab-35aef392ea70__download_file_content
  - mcp__af9311f4-61e6-4373-89b5-815b4f05149e__get_thread
  - mcp__computer-use__screenshot
  - mcp__computer-use__left_click
  - mcp__computer-use__type
  - mcp__computer-use__open_application
  - mcp__computer-use__request_access
  - mcp__computer-use__scroll
  - mcp__computer-use__key
  - mcp__computer-use__double_click
  - Bash
---

You are processing pending invoice approvals for Barry Halag (baruch.halag@gmail.com).

## Overview
This command handles two types of items:
1. **Unknown suppliers** — invoices held in pending-review.json because Claude hasn't seen this supplier before
2. **Blocked supplier exceptions** — invoices that were auto-skipped (supplier is on blocked list) but Barry saw them in the digest and wants to upload this specific one anyway

Barry uses this command after reviewing the daily digest email.

## Steps

### 1. Load Pending Items
Search Google Drive for "invoice-scanner pending-review" and read pending-review.json.
Also read processed-log.json to check if Barry mentions a specific blocked invoice he wants to re-open.

If the items array is empty AND Barry didn't mention a specific blocked supplier: tell user "Nothing pending — all clear! ✅" and stop.

**If Barry mentions a blocked supplier exception** (e.g. "I want to upload the Partner invoice from today"):
- Find the matching thread in processed-log.json (outcome="skipped_blocked_supplier")
- Treat it as a one-time upload (Choice 2 below) — do NOT change the supplier's blocked status
- After upload, update processed-log.json outcome to "uploaded_exception"

### 2. Load Supporting Data
Also load:
- supplier-database.json (to update with user's decisions)
- processed-log.json (to mark items as processed)
- cpa-upload-script.json (to run uploads if user approves)

### 3. Present Each Pending Item
For each item in pending-review.json with status="awaiting_decision", present it one at a time:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 INVOICE REVIEW — Item [N] of [Total]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Supplier:  [supplier_detected]
Email:     [sender_email]
Subject:   [subject]
Amount:    [amount] [currency]
Date:      [detected_at date]
File:      [attachment_name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

What would you like to do?
[1] Upload now + always auto-approve this supplier (tax-deductible)
[2] Upload now + ask me again next time (one-off)
[3] Skip this invoice + block supplier permanently (not deductible)
[4] Skip this invoice only — decide about supplier later
```

Wait for user's response (they can type 1, 2, 3, or 4, or write their choice).

### 4. Execute User's Choice

**Choice 1 — Upload + auto-approve supplier:**
- Download the attachment from Gmail (get_thread, extract attachment, save to /tmp/).
- Upload to CPA app using cpa-upload-script.json steps via computer-use.
- On success:
  - Ask user: "What category is this expense? (communications / software_subscriptions / office_supplies / professional_services / travel_business / equipment / insurance_business / other_business)"
  - Add supplier to supplier-database.json:
    ```json
    {
      "normalized_name": "<normalized>",
      "auto_approve": true,
      "tax_deductible": true,
      "deductible_category": "<user_answer>",
      "first_seen": "<today>",
      "last_decision": "<today>",
      "decision_made_by": "user",
      "notes": "",
      "email_patterns": ["<@domain_from_sender_email>"]
    }
    ```
  - Update processed-log.json: outcome="uploaded", remove from pending-review.json.

**Choice 2 — Upload once, ask again next time:**
- Download and upload to CPA app (same as choice 1).
- Do NOT add to supplier-database.json.
- Update processed-log.json: outcome="uploaded_one_time".
- Remove from pending-review.json.

**Choice 3 — Skip + block supplier permanently:**
- Add supplier to supplier-database.json with auto_approve=false, tax_deductible=false.
- Update processed-log.json: outcome="skipped_user_rejected".
- Remove from pending-review.json.
- Confirm: "Supplier [name] blocked. Future invoices from them will be skipped automatically."

**Choice 4 — Skip this invoice only:**
- Update processed-log.json: outcome="skipped_deferred".
- Keep item in pending-review.json (will reappear in next digest).
- Confirm: "Skipped for now. This invoice will appear again in tomorrow's digest."

### 5. Continue to Next Item
Move to the next pending item. Repeat until all are processed.

### 6. Save All Files
After processing all items, write updated files back to Google Drive:
- supplier-database.json (if any choices 1 or 3 were made)
- processed-log.json (always update)
- pending-review.json (always update)

### 7. Confirm Completion
Report summary:
```
✅ Done reviewing pending invoices.
   Uploaded: N
   Auto-approved for future: N
   Blocked: N
   Deferred: N
```

## Notes
- If a CPA upload fails, log outcome="upload_failed_retry" in processed-log.json and keep in pending-review.json.
- Always save Google Drive files immediately after each decision, in case the session is interrupted.
- The supplier's email domain (e.g., @supplier.co.il) is automatically extracted from sender_email and added to email_patterns.
