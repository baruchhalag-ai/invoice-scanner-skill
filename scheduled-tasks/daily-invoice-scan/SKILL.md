# Daily Invoice Scanner — Scheduled Task

You are running the automated weekly invoice scan for Barry Halag (baruch.halag@gmail.com).
This task runs every Monday at 08:30 Israel time.

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
4. Load Gmail credentials from local file: `~/.claude/invoice-scanner-secrets.json`
   - Fields: gmail_user, gmail_app_password, cpa_email
   - If file missing or unreadable: STOP. Create a Gmail draft:
     Subject: "Invoice Scanner — SETUP REQUIRED"
     Body: "Credentials file missing: ~/.claude/invoice-scanner-secrets.json. Please open Claude to reconfigure."
   - Then stop execution.

### PHASE 2: GMAIL SCAN

Run these searches using mcp__af9311f4__search_threads.
IMPORTANT: All queries use `in:anywhere` to search ALL folders including Spam, Promotions, All Mail.
Queries do NOT require has:attachment — this catches notification emails from suppliers who send
"invoice ready" emails without attaching the file.

**Search 1 — Hebrew invoice keywords:**
`in:anywhere (חשבונית OR קבלה OR "חשבון מס" OR "חשבונית מס") newer_than:8d`

**Search 2 — English invoice keywords:**
`in:anywhere (invoice OR receipt OR "tax invoice" OR billing) newer_than:8d`

**Search 3 — Monthly summary / billing keywords (catches suppliers like פנגו who use neither):**
`in:anywhere (סיכום OR "פירוט חיובים" OR "חשבון חודשי" OR "תקופת חיוב" OR "statement") newer_than:8d`

**Search 4 — Known sender domains from supplier-database.json email_patterns:**
Build: `in:anywhere (from:@domain1.com OR from:@domain2.com) newer_than:8d`
Skip if no email_patterns exist.

Combine all results. De-duplicate by thread ID.
Remove any thread ID that already exists in processed-log.json.processed_emails.
Result: candidate list of new invoice threads.

### ⚠️ PHASE 2B: IMAP SCAN FOR RELAY-SENDER SUPPLIERS
Some suppliers send via third-party email relays (e.g. meser1send.com) which makes their emails
INVISIBLE to Gmail API search — `from:` queries fail silently and the emails never appear in results.

For every supplier in supplier-database.json that has `"imap_search_note"` set:
1. Use Python IMAP to search: `SINCE '{today-8d}' FROM '{imap_from_filter}'`
2. For each message found, decode the subject and check if it matches `subject_pattern`.
3. If it does AND the thread ID is not in processed-log.json → add to candidate list.

```python
import imaplib, email, json, datetime

with open('/Users/barryhalag/.claude/invoice-scanner-secrets.json') as f:
    secrets = json.load(f)

mail = imaplib.IMAP4_SSL("imap.gmail.com")
mail.login(secrets['gmail_user'], secrets['gmail_app_password'])
mail.select('"[Gmail]/All Mail"')

since_date = (datetime.date.today() - datetime.timedelta(days=8)).strftime("%d-%b-%Y")
# e.g. for פנגו: imap_from_filter = "pango"
res, data = mail.uid("search", None, f'SINCE "{since_date}" FROM "{imap_from_filter}"')
if data[0]:
    for uid in data[0].split():
        res2, msg_data = mail.uid("fetch", uid, "(X-GM-THRID X-GM-MSGID ENVELOPE)")
        # Extract thread ID and subject, check subject_pattern, add to candidates if new
        ...
mail.logout()
```

Currently known relay-sender suppliers: **פנגו** (FROM "pango", subject pattern "סיכום חודש")

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

### ⚠️ PRE-SCANNER INVOICES — EXTRA CHECK REQUIRED
The processed-log only tracks uploads made through this scanner (from 2026-04-29 onward).
Any invoice uploaded manually before that date will NOT appear in the log.

**Rule:** When manually uploading any invoice whose email arrived before 2026-04-29,
FIRST check חדד לוי (https://hadadlevi.account-ant.com/..../documents) to confirm it is
not already there. Only upload if it is absent.
Failure to check will create duplicates that Barry must manually delete from the CPA app.

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

#### Upload Method: Email Forwarding via IMAP + SMTP
The CPA system accepts invoices sent by email to: acc+052694569@account-ant.com
Documents appear automatically in the system with status "ממתין" (pending CPA review).
No web browser or app interaction needed.

For each item in upload queue, run this Python script via Bash:

```python
import imaplib, smtplib, email, json
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders

# Load credentials
with open('/Users/barryhalag/.claude/invoice-scanner-secrets.json') as f:
    secrets = json.load(f)
GMAIL_USER = secrets['gmail_user']
APP_PASS = secrets['gmail_app_password']
CPA_EMAIL = secrets['cpa_email']

# attachment_filter: from supplier entry, e.g. {"type": "filename_prefix", "prefix": "Invoice-"}
# thread_id: Gmail thread ID (hex string)

def upload_invoice(thread_id, attachment_filter=None):
    # Connect via IMAP
    mail = imaplib.IMAP4_SSL("imap.gmail.com")
    mail.login(GMAIL_USER, APP_PASS)
    mail.select('"[Gmail]/All Mail"')

    # Find message by Gmail thread ID
    dec_id = int(thread_id, 16)
    res, data = mail.uid("search", None, f'X-GM-THRID {dec_id}')
    if res != "OK" or not data[0]:
        mail.logout()
        return {"ok": False, "reason": "thread not found"}

    uid = data[0].split()[-1]
    res, msg_data = mail.uid("fetch", uid, "(RFC822)")
    raw = msg_data[0][1]
    msg = email.message_from_bytes(raw)
    subject = msg.get("Subject", "Invoice")
    mail.logout()

    # Find the correct PDF attachment
    pdf_found = None
    for part in msg.walk():
        fn = part.get_filename() or ""
        ct = part.get_content_type()
        if not ("pdf" in ct or fn.lower().endswith(".pdf")):
            continue
        # Apply attachment_filter if present
        if attachment_filter and attachment_filter.get("type") == "filename_prefix":
            if not fn.startswith(attachment_filter["prefix"]):
                continue
        pdf_found = (fn, part.get_payload(decode=True))
        break

    if not pdf_found:
        return {"ok": False, "reason": "no matching PDF attachment"}

    fn, pdf_data = pdf_found

    # Send to CPA via SMTP
    fwd = MIMEMultipart()
    fwd["From"] = GMAIL_USER
    fwd["To"] = CPA_EMAIL
    fwd["Subject"] = subject
    part = MIMEBase("application", "pdf")
    part.set_payload(pdf_data)
    encoders.encode_base64(part)
    part.add_header("Content-Disposition", f'attachment; filename="{fn}"')
    fwd.attach(part)

    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(GMAIL_USER, APP_PASS)
        smtp.send_message(fwd)

    return {"ok": True, "filename": fn, "subject": subject}
```

On success:
- Add to processed-log.json.processed_emails:
  ```json
  {
    "supplier": "<supplier_name>",
    "sender_email": "<email>",
    "subject": "<subject>",
    "invoice_number": "<if found>",
    "outcome": "uploaded",
    "processed_at": "<ISO datetime>",
    "attachment_name": "<filename>",
    "method": "email_forward_smtp",
    "retry_count": 0
  }
  ```
- Also log to invoice_numbers and billed_months keys in processed-log.json.
- Increment processed-log.json.stats.total_uploaded and total_processed.

On failure:
- If retry_count < 3: set outcome="upload_failed_retry", increment retry_count.
- If retry_count >= 3: set outcome="upload_failed_permanent".
- Add to processed-log.json and include in digest ⚠️ section.

### PHASE 4 — SUPPLEMENT: NON-ATTACHMENT RETRIEVAL METHODS

For suppliers where retrieval_method is NOT "email_attachment", use the appropriate method below
before uploading via SMTP. Once the PDF is in /tmp/, upload exactly as in Phase 4.

#### Method: "link_in_email" (e.g. רייזאפ)
The email HTML body contains a tracking link that redirects to a direct PDF URL (no login).

```python
import imaplib, email, re, urllib.request, json, smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders

def retrieve_link_in_email(thread_id, supplier_name, date_str):
    """
    1. Fetch email via IMAP
    2. Extract first href from HTML body
    3. Follow redirect with urllib to get final PDF URL
    4. Download PDF bytes
    5. Send to CPA via SMTP
    """
    with open('/Users/barryhalag/.claude/invoice-scanner-secrets.json') as f:
        secrets = json.load(f)
    GMAIL_USER = secrets['gmail_user']
    APP_PASS = secrets['gmail_app_password']
    CPA_EMAIL = secrets['cpa_email']

    # Step 1: Fetch email
    mail = imaplib.IMAP4_SSL("imap.gmail.com")
    mail.login(GMAIL_USER, APP_PASS)
    mail.select('"[Gmail]/All Mail"')
    dec_id = int(thread_id, 16)
    res, data = mail.uid("search", None, f'X-GM-THRID {dec_id}')
    uid = data[0].split()[-1]
    res, msg_data = mail.uid("fetch", uid, "(RFC822)")
    raw = msg_data[0][1]
    msg = email.message_from_bytes(raw)
    subject = msg.get("Subject", "Invoice")
    mail.logout()

    # Step 2: Extract first href from HTML body
    html_body = ""
    for part in msg.walk():
        if part.get_content_type() == "text/html":
            html_body = part.get_payload(decode=True).decode('utf-8', errors='replace')
            break
    hrefs = re.findall(r'href=["\']([^"\']+)["\']', html_body)
    # Filter out unsubscribe/mailto links
    invoice_links = [h for h in hrefs if 'unsubscribe' not in h.lower() and not h.startswith('mailto')]
    if not invoice_links:
        return {"ok": False, "reason": "no link found in email HTML"}
    tracking_url = invoice_links[0]

    # Step 3: Follow redirect to get final PDF URL
    req = urllib.request.Request(tracking_url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        final_url = resp.url
        pdf_data = resp.read()

    filename = f"{supplier_name}_{date_str}.pdf"

    # Step 4: Send to CPA via SMTP
    fwd = MIMEMultipart()
    fwd["From"] = GMAIL_USER
    fwd["To"] = CPA_EMAIL
    fwd["Subject"] = subject
    part = MIMEBase("application", "pdf")
    part.set_payload(pdf_data)
    encoders.encode_base64(part)
    part.add_header("Content-Disposition", f'attachment; filename="{filename}"')
    fwd.attach(part)
    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(GMAIL_USER, APP_PASS)
        smtp.send_message(fwd)

    return {"ok": True, "filename": filename, "subject": subject, "source_url": final_url}
```

#### Method: "link_in_email_js_spa" (e.g. פנגו)
The email is found via **IMAP** (Gmail API cannot find it — relay sender).
The email contains TWO types of URLs — only one is the actual invoice:

⚠️ **CRITICAL — TWO URLS, USE ONLY THE 4500.co.il ONE:**
- `admin.pango.co.il/driver/ProofOfPaymentDetailed.aspx?Statement={uuid}` — ❌ **WRONG**: this is a
  detail-breakdown page showing individual parking sessions, NOT a formal חשבונית. Do NOT use this.
- `mcpsmartphonews.4500.co.il/ShowStatementRedirect/Default.aspx?UniqueID={uuid}&StatementKind=0`
  — ✅ **CORRECT**: this redirects to the actual signed invoice PDF at
  `https://invoice.pango.co.il/YYYY/{id}/{uuid}_...Sign.pdf`. Download directly — no login needed.

```python
import imaplib, email, re, urllib.request, json, smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.base import MIMEBase
from email import encoders

def retrieve_pango_invoice(imap_uid, date_str):
    with open('/Users/barryhalag/.claude/invoice-scanner-secrets.json') as f:
        secrets = json.load(f)
    GMAIL_USER = secrets['gmail_user']
    APP_PASS = secrets['gmail_app_password']
    CPA_EMAIL = secrets['cpa_email']

    # Step 1: Fetch email via IMAP (uid is int from IMAP search)
    mail = imaplib.IMAP4_SSL("imap.gmail.com")
    mail.login(GMAIL_USER, APP_PASS)
    mail.select('"[Gmail]/All Mail"')
    res, msg_data = mail.uid("fetch", str(imap_uid).encode(), "(RFC822)")
    raw = msg_data[0][1]
    msg = email.message_from_bytes(raw)
    subject = msg.get("Subject", "Invoice")
    mail.logout()

    # Step 2: Extract all URLs from HTML body
    html_body = ""
    for part in msg.walk():
        if part.get_content_type() == "text/html":
            html_body = part.get_payload(decode=True).decode('utf-8', errors='replace')
            break
    all_urls = re.findall(r'https?://[^\s"\'<>]+', html_body)

    # Step 3: Find the 4500.co.il StatementKind=0 URL — this gives the real invoice PDF
    invoice_redirect_url = None
    for u in all_urls:
        if '4500.co.il' in u and 'StatementKind=0' in u:
            invoice_redirect_url = u.rstrip(').,;')
            break
    if not invoice_redirect_url:
        return {"ok": False, "reason": "4500.co.il StatementKind=0 URL not found in email"}

    # Step 4: Follow redirect → direct PDF at invoice.pango.co.il
    req = urllib.request.Request(invoice_redirect_url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=15) as resp:
        final_pdf_url = resp.url
        pdf_data = resp.read()

    if not final_pdf_url.endswith('.pdf'):
        return {"ok": False, "reason": f"Redirect did not lead to PDF: {final_pdf_url}"}

    filename = f"pango_{date_str}_invoice.pdf"

    # Step 5: Send to CPA via SMTP
    fwd = MIMEMultipart()
    fwd["From"] = GMAIL_USER
    fwd["To"] = CPA_EMAIL
    fwd["Subject"] = subject
    part = MIMEBase("application", "pdf")
    part.set_payload(pdf_data)
    encoders.encode_base64(part)
    part.add_header("Content-Disposition", f'attachment; filename="{filename}"')
    fwd.attach(part)
    with smtplib.SMTP_SSL("smtp.gmail.com", 465) as smtp:
        smtp.login(GMAIL_USER, APP_PASS)
        smtp.send_message(fwd)

    return {"ok": True, "filename": filename, "source_url": final_pdf_url}
```

#### Method: "website_notification" with JavaScript SPA (e.g. ארנונה MAST)
The email contains a URL to a JavaScript-rendered page that embeds the PDF in an iframe.
Requires Chrome MCP to render the page and extract the PDF URL. NO LOGIN NEEDED.

```python
# Step 1: Extract the bill-viewer URL from the email HTML (same as link_in_email step 1-2)
# The URL pattern for MAST: https://mast.co.il/bill-viewer/...

# Step 2: Use Chrome MCP to navigate and extract the PDF URL
# mcp__Claude_in_Chrome__navigate(tabId, url=bill_viewer_url)
# Wait 4 seconds for JS to render
# mcp__Claude_in_Chrome__javascript_tool(tabId, text="""
#   Array.from(document.querySelectorAll('iframe, embed, object'))
#     .map(el => el.src || el.data || '')
#     .find(src => src.includes('.pdf') || src.includes('blob.core.windows.net'))
# """)
# This returns the Azure Blob SAS URL (time-limited but fresh each page load)

# Step 3: Download the PDF from the extracted URL via Python
# import urllib.request
# req = urllib.request.Request(pdf_url, headers={'User-Agent': 'Mozilla/5.0'})
# pdf_data = urllib.request.urlopen(req, timeout=15).read()

# Step 4: Send to CPA via SMTP (same as email_attachment method)
```

Note: The Azure SAS token in the PDF URL expires (usually within hours), but the
bill-viewer URL in the email is long-lived. Always navigate fresh to get a valid SAS URL.

### PHASE 4B: VERIFY UPLOADS AT חדד לוי רואי חשבון

Run this after ALL uploads in Phase 4A are complete (skip if upload queue was empty).

Purpose:
1. Confirm every uploaded invoice actually appears in the CPA system.
2. Detect if any invoice was accidentally uploaded twice (duplicate).

#### Key principle — targeted search, NOT full page read:
We already know exactly which invoice numbers we just uploaded (from the upload queue).
So instead of reading the entire document list (expensive), we search for each specific
invoice number on the page. This uses minimal tokens.

#### Steps:

1. Navigate to the CPA documents page using Chrome MCP:
   URL: https://hadadlevi.account-ant.com/7914A263-C7D5-4C3E-A0C5-2D836B6BB7D7/documents
   Wait 5 seconds for the page to load.

2. For each invoice just uploaded, extract its search key:
   - If invoice_number is known (e.g. "QPD7ZLKP-0006") → use that as search key.
   - If no invoice number → use the filename without extension (e.g. "חשבונית-2026-04").

3. For each search key, run:
   mcp__Claude_in_Chrome__find with query = the search key string.
   Count the number of matching elements returned.

   - Count = 0 → NOT FOUND. Add to digest ⚠️:
     "⚠️ [supplier] | [invoice_number] — שולח אך לא נמצא בחדד לוי. ייתכן עיכוב. בדוק ידנית."
     Update processed-log.json outcome to "upload_sent_unverified".
   - Count = 1 → VERIFIED ✅. Update processed-log.json outcome to "uploaded_verified".
   - Count ≥ 2 → DUPLICATE ⚠️. Add to digest:
     "⚠️ כפילות זוהתה בחדד לוי: [supplier] | [invoice_number] — מופיע [count] פעמים. בדוק ידנית."
     Update processed-log.json outcome to "uploaded_duplicate_detected".

4. No screenshot needed unless a duplicate or missing invoice is found.

#### Important notes:
- If navigation fails (not logged in / session expired): log outcome="verification_skipped_session_expired"
  and add to digest: "⚠️ לא ניתן לאמת העלאות — תוקף ההתחברות לחדד לוי פג. בדוק ידנית."
- Do NOT block the rest of the run if verification fails — continue to Phase 5.
- If an invoice is not found, it is "unverified" (not "failed") — email forwarding can have a short delay.

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

### PHASE 8: SEND PHONE NOTIFICATION
Send a push notification to Barry's phone via ntfy.

Load ntfy_channel from ~/.claude/invoice-scanner-secrets.json.

Run via Bash:
```bash
curl -s \
  -H "Title: סריקת חשבוניות שבועית 🧾" \
  -H "Priority: default" \
  -H "Tags: white_check_mark" \
  -d "[MESSAGE]" \
  https://ntfy.sh/[ntfy_channel]
```

Build [MESSAGE] based on results:
- If uploaded > 0 and pending = 0:
  "✅ הועלו {uploaded} חשבוניות אוטומטית. אין פעולה נדרשת."
- If pending > 0:
  "✅ הועלו {uploaded} חשבוניות. ⏳ {pending} ממתינות לאישורך — פתח Claude ורשום /invoice-approve"
- If failed > 0:
  Add: " ⚠️ {failed} נכשלו בהעלאה."
- If nothing found:
  "🔍 לא נמצאו חשבוניות חדשות השבוע."

### PHASE 9: DONE
Log: "Weekly invoice scan complete: [uploaded] uploaded, [pending] pending review, [skipped] skipped."
