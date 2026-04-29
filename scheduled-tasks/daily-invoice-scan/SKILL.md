# Daily Invoice Scanner — Scheduled Task

You are running the automated daily invoice scan for Barry Halag (baruch.halag@gmail.com).
This task runs every day at 08:30 Israel time.

Your job: scan Gmail for new invoices, upload approved ones to the CPA app "חדד לוי רואי חשבון",
and create a Gmail draft summarizing what was processed and what needs user review.

## Data Files Location
All data is in Google Drive, folder "My Drive/Claude/invoice-scanner/".
Use the Google Drive MCP (mcp__69f0d116...) to read and write files.
Use search_files to locate each file, then read_file_content.

Files:
- supplier-database.json — approved/blocked suppliers
- processed-log.json — deduplication log (thread IDs already processed)
- pending-review.json — unknown suppliers awaiting user decision
- cpa-upload-script.json — recorded computer-use steps for CPA app
- run-log.json — daily run history

## Gmail Account
baruch.halag@gmail.com
Use mcp__af9311f4... tools for Gmail operations.

---

## EXECUTE THIS FULL SEQUENCE

### PHASE 1: LOAD STATE
1. Read supplier-database.json from Google Drive.
2. Read processed-log.json from Google Drive.
3. Read pending-review.json from Google Drive.
4. Read cpa-upload-script.json from Google Drive.
   - If teaching_complete is false or missing: STOP. Create a Gmail draft to baruch.halag@gmail.com:
     Subject: "Invoice Scanner — SETUP REQUIRED"
     Body: "The CPA app teaching session has not been completed. Please open Claude and run /invoice-scanner teach."
   - Then stop execution.

### PHASE 2: GMAIL SCAN
Run these 3 searches using mcp__af9311f4__search_threads.
IMPORTANT: All queries use `in:anywhere` to search ALL folders including Spam, Promotions, All Mail.
Queries do NOT require has:attachment — this catches notification emails from suppliers like
כביש 6 who send "invoice ready" emails without attaching the file.

1. Query: `in:anywhere (חשבונית OR קבלה OR "חשבון מס" OR "חשבונית מס") newer_than:2d`
2. Query: `in:anywhere (invoice OR receipt OR "tax invoice" OR billing) newer_than:2d`
3. Query: Build a from-domain query from email_patterns in supplier-database.json.
   Example: `in:anywhere (from:@partner.co.il OR from:@cellcom.co.il) newer_than:2d`
   Skip this search if no email_patterns exist yet.

Combine all results. De-duplicate by thread ID.
Remove any thread ID that already exists as a key in processed-log.json.processed_emails.
Result: candidate list of new invoice threads.

## DEDUPLICATION — THREE KEYS (run before every upload)
1. Extract invoice number from email subject or attachment filename using regex:
   - Hebrew: חשבונית\s*(?:מס)?\s*(\d+) | מספר\s*(\d+) | קבלה\s*(\d+)
   - English: invoice\s*#?\s*(\d+) | inv[-\s#]?(\d+) | #(\d{4,})
2. Check ALL three keys — skip if ANY matches:
   - processed_emails[thread_id]
   - invoice_numbers["supplier-INV-number"] (if invoice number found)
   - billed_months["supplier-YYYY-MM"] (fallback if no invoice number)
3. If invoice number not found AND billed_months key already exists:
   → DO NOT upload. Add to digest: "⚠️ Second invoice from [supplier] this month — invoice number not found. Please verify manually."
4. After upload, log ALL applicable keys to processed-log.json.

### PHASE 3: CLASSIFY CANDIDATES
For each candidate thread ID:
1. Call mcp__af9311f4__get_thread to get full thread details.
2. Extract:
   - sender_display_name: the "From:" display name
   - sender_email: the "From:" email address
   - subject: email subject
   - invoice_date: from date header
   - attachment_names: list of attached file names
3. Normalize the supplier name:
   - Lowercase everything
   - Strip: בע"מ, בעמ, פע"מ, בע"ם, ltd, inc, co., corp, gmbh (and trailing punctuation)
   - Strip leading/trailing whitespace
4. Look up in supplier-database.json:
   a. Check if normalized_name matches any supplier's normalized_name field (exact string match)
   b. If no match: check if sender_email domain (@domain.com) matches any supplier's email_patterns array
   c. If still no match: classify as UNKNOWN
5. Route:
   - Supplier found AND auto_approve=true → upload queue
   - Supplier found AND auto_approve=false → skip list (blocked)
   - UNKNOWN → pending list

### PHASE 4: UPLOAD AUTO-APPROVED INVOICES
For each item in upload queue:
1. From the get_thread result, identify the attachment.
   Save it to /tmp/invoice_[thread_id].[ext] using Bash.
2. Load the computer-use script steps from cpa-upload-script.json.
3. Execute each step using mcp__computer-use tools:
   - open_application: open "חדד לוי רואי חשבון"
   - Replay recorded steps, substituting variable fields:
     * file_path: /tmp/invoice_[thread_id].[ext]
     * date: invoice_date (formatted per field_mapping)
     * amount: extracted from subject or body if possible
     * supplier: sender_display_name
     * category: supplier's deductible_category from supplier-database.json
4. After each upload step, take a screenshot to verify.
5. On success:
   - Add to processed-log.json.processed_emails:
     ```json
     {
       "supplier": "<supplier_name>",
       "sender_email": "<email>",
       "subject": "<subject>",
       "outcome": "uploaded",
       "processed_at": "<ISO datetime>",
       "attachment_name": "<filename>",
       "retry_count": 0
     }
     ```
   - Increment processed-log.json.stats.total_uploaded and total_processed
6. On failure:
   - If existing retry_count < 3: set outcome="upload_failed_retry", increment retry_count
   - If retry_count >= 3: set outcome="upload_failed_permanent"
   - Add to processed-log.json
   - Clean up /tmp/ file

### PHASE 5: HANDLE PENDING AND SKIPPED
For each item in pending list (unknown suppliers):
- Add to pending-review.json.items:
  ```json
  {
    "id": "pending-<YYYYMMDD>-<sequence>",
    "gmail_thread_id": "<thread_id>",
    "detected_at": "<ISO datetime>",
    "supplier_detected": "<sender_display_name>",
    "sender_email": "<email>",
    "subject": "<subject>",
    "attachment_name": "<filename>",
    "status": "awaiting_decision"
  }
  ```
- Increment processed-log.json.stats.total_pending

For each item in skip list (blocked suppliers):
- Add to processed-log.json.processed_emails with outcome="skipped_blocked_supplier"
- Increment processed-log.json.stats.total_skipped
- Keep a separate "skipped_list" in memory for the digest (include supplier, subject, amount)

### PHASE 6: CREATE GMAIL DRAFT DIGEST
Compose a Gmail draft to baruch.halag@gmail.com.

Subject: "Invoice Scanner — Daily Digest [DD/MM/YYYY]"

Body (in Hebrew and English):
```
שלום Barry,

סיכום סריקת החשבוניות היומית:

✅ הועלו אוטומטית: [N] חשבוניות
[For each uploaded: - [supplier_name] | [subject] | [amount if detected]]

⏳ ממתינים לאישורך (ספקים לא מוכרים): [N] חשבוניות
[For each pending: - [supplier_detected] | [subject] | [amount if detected]]
לטיפול: פתח Claude ורשום /invoice-approve

🚫 דולגו (ספקים חסומים) — לידיעתך בלבד: [N]
[For each skipped: - [supplier_name] | [subject] | [amount if detected]]
אם חשבונית מסוימת כן ניתנת לניכוי בניגוד לכלל, פתח Claude ורשום /invoice-approve

⚠️ נכשלו בהעלאה (ינסה שוב): [N]
[For each failed: - [supplier_name] | [subject]]

תאריך הרצה: [datetime]
```

IMPORTANT: Blocked suppliers MUST always appear in the digest, even if auto-skipped.
This allows Barry to spot exceptions — e.g. a normally-personal supplier who sent a
legitimate business invoice. If he wants to upload a skipped invoice, he runs /invoice-approve.

If ALL counts are 0 (nothing found): Subject "Invoice Scanner — Nothing New [DATE]", Body: "No new invoice emails found today."

Use mcp__af9311f4__create_draft to create the draft.

### PHASE 7: SAVE STATE
Write updated files back to Google Drive (create_file, overwriting existing):
1. processed-log.json — always write
2. pending-review.json — always write
3. run-log.json — append run entry:
   ```json
   {
     "run_at": "<ISO datetime>",
     "candidates_found": <N>,
     "uploaded": <N>,
     "skipped": <N>,
     "pending": <N>,
     "failed": <N>
   }
   ```

### PHASE 8: DONE
Log: "Daily invoice scan complete: [uploaded] uploaded, [pending] pending review, [skipped] skipped."
