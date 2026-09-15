---
name: daily-invoice-scan
description: Weekly invoice scan (Mondays 08:30) for baruch.halag@gmail.com → CPA upload
---

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

#### Determine the scan window
Read `stats.last_scan_date` from processed-log.json:
- If null (first run): use `after:2020/01/01` — scan everything from the beginning.
- Otherwise: use `after:YYYY/MM/DD` where the date is the value of `last_scan_date`
  formatted with slashes instead of dashes.
  Example: `"last_scan_date": "2026-09-08"` → `after:2026/09/08`

Run these 3 searches using mcp__af9311f4__search_threads.
Replace `[AFTER_DATE]` below with the `after:YYYY/MM/DD` value determined above.

1. Query: `in:anywhere (חשבונית OR קבלה OR "חשבון מס" OR "חשבונית מס") [AFTER_DATE]`
2. Query: `in:anywhere (invoice OR receipt OR "tax invoice" OR billing) [AFTER_DATE]`
3. Query: Build a from-domain query from email_patterns in supplier-database.json.
   Example: `in:anywhere (from:@partner.co.il OR from:@cellcom.co.il) [AFTER_DATE]`
   Skip this search if no email_patterns exist yet.

Note: queries do NOT require has:attachment — some suppliers (e.g. פנגו, כביש 6) send notification emails without attachments. The retrieval_method field in supplier-database.json determines how to get the actual PDF.

Combine all results. De-duplicate by thread ID.
Remove any thread ID that already exists as a key in processed-log.json.processed_emails.
Result: candidate list of new invoice threads.

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
Upload method: SMTP email forwarding (NO computer-use, NO native app interaction).
Credentials: load from ~/.claude/invoice-scanner-secrets.json
  - gmail_user: baruch.halag@gmail.com
  - gmail_app_password: the app password
  - cpa_email: acc+052694569@account-ant.com

The CPA system automatically ingests any PDF emailed to cpa_email. No manual steps needed.

For each item in upload queue, determine retrieval_method from supplier-database.json, then:

**retrieval_method = "email_attachment"**: Run this Python via Bash:
```python
import imaplib, smtplib, email, json
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders

with open('/Users/barryhalag/.claude/invoice-scanner-secrets.json') as f:
    s = json.load(f)
GMAIL_USER = s['gmail_user']
APP_PASS = s['gmail_app_password']
CPA_EMAIL = s['cpa_email']

mail = imaplib.IMAP4_SSL("imap.gmail.com")
mail.login(GMAIL_USER, APP_PASS)
mail.select('"[Gmail]/All Mail"')
dec_id = int(thread_id, 16)
res, data = mail.uid("search", None, f'X-GM-THRID {dec_id}')
uid = data[0].split()[-1]
res, msg_data = mail.uid("fetch", uid, "(RFC822)")
msg = email.message_from_bytes(msg_data[0][1])
mail.logout()

# Find PDF attachment (apply attachment_filter if set for this supplier)
pdf_found = None
for part in msg.walk():
    fn = part.get_filename() or ""
    ct = part.get_content_type()
    if "pdf" in ct or fn.lower().endswith(".pdf"):
        # Apply attachment_filter if defined (e.g. prefix "Invoice-")
        pdf_found = (fn, part.get_payload(decode=True))
        break

if pdf_found:
    fn, pdf_data = pdf_found
    fwd = MIMEMultipart()
    fwd["From"] = GMAIL_USER
    fwd["To"] = CPA_EMAIL
    fwd["Subject"] = msg.get("Subject", "Invoice")
    part = MIMEBase("application", "pdf")
    part.set_payload(pdf_data)
    encoders.encode_base64(part)
    part.add_header("Content-Disposition", f'attachment; filename="{fn}"')
    fwd.attach(part)
    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(GMAIL_USER, APP_PASS)
        smtp.send_message(fwd)
    print(f"UPLOADED: {fn}")
else:
    print("ERROR: no PDF found")
```

**retrieval_method = "link_in_email"** (פזגז, רייזאפ): Follow the website_script steps in supplier-database.json to download the PDF to /tmp/, then send it to CPA_EMAIL via SMTP (same as above but attach the /tmp/ file).

**retrieval_method = "link_in_email_direct_pdf"** (פנגו): Follow the website_script steps:
  1. IMAP search SINCE/BEFORE with FROM 'pango' to find the thread
  2. Extract the 4500.co.il/ShowStatementRedirect URL with StatementKind=0 (NOT ProofOfPaymentDetailed.aspx)
  3. urllib follow redirect → download signed PDF from invoice.pango.co.il
  4. Send to CPA_EMAIL via SMTP

**retrieval_method = "website_notification"** (כביש 6, ארנונה, מים): These require SMS verification — SKIP for now, add to pending list with note "requires interactive SMS verification".

**retrieval_method = "email_html_to_pdf"** (Google Play): Print the HTML email body to PDF using Chrome headless, then send to CPA_EMAIL via SMTP.

After each upload attempt:
- On success: add to processed-log with outcome="uploaded", method="email_forward_smtp"
- On failure: set outcome="upload_failed_retry" if retry_count < 3, else "upload_failed_permanent"

### PHASE 5: HANDLE PENDING AND SKIPPED
For each item in pending list (unknown suppliers):
- Add to pending-review.json.items with status="awaiting_decision"
- Increment processed-log.json.stats.total_pending

For each item in skip list (blocked suppliers):
- Add to processed-log.json.processed_emails with outcome="skipped_blocked_supplier"
- Increment processed-log.json.stats.total_skipped
- Keep in a skipped_list for the digest (supplier, subject, amount)

### PHASE 6: CREATE GMAIL DRAFT DIGEST
Compose a Gmail draft to baruch.halag@gmail.com.

Subject: "Invoice Scanner — Daily Digest [DD/MM/YYYY]"

Body:
שלום Barry,

סיכום סריקת החשבוניות היומית:

✅ הועלו אוטומטית: [N] חשבוניות
[For each uploaded: - supplier | subject | amount if detected]

⏳ ממתינים לאישורך (ספקים לא מוכרים): [N] חשבוניות
[For each pending: - supplier | subject | amount if detected]
לטיפול: פתח Claude ורשום /invoice-approve

🚫 דולגו (ספקים חסומים) — לידיעתך בלבד: [N]
[For each skipped: - supplier | subject | amount if detected]
אם חשבונית מסוימת כן ניתנת לניכוי, פתח Claude ורשום /invoice-approve

⚠️ נכשלו בהעלאה (ינסה שוב): [N]
[For each failed: - supplier | subject]

תאריך הרצה: [datetime]

IMPORTANT: Blocked suppliers MUST always appear in the digest. Barry reviews the 🚫 section to catch exceptions where a blocked supplier sent a legitimate deductible invoice.

If ALL counts are 0: Subject "Invoice Scanner — Nothing New [DATE]", Body: "No new invoice emails found today."

Use mcp__af9311f4__create_draft to create the draft.

### PHASE 7: SAVE STATE
Before writing:
- Set `stats.last_scan_date` in processed-log.json to today's date (format: "YYYY-MM-DD").
  This is the cursor — Phase 2 of the next run will search `after:` this date.
- Set `stats.last_run` to the current ISO datetime.

Write updated files back to Google Drive using create_file + trash_file (create new, trash old):
1. processed-log.json — always write
2. pending-review.json — always write
3. run-log.json — append run entry: {run_at, scan_from, candidates_found, uploaded, skipped, pending, failed}
   `scan_from` = the `after:` date used in this run's Phase 2.

Use parentId "1CqVGyQizqp5ZxTdy7lke3CAcd7qH7SJm" (the invoice-scanner folder) for all three files.

### PHASE 8: DONE
Log: "Daily invoice scan complete: [uploaded] uploaded, [pending] pending review, [skipped] skipped."