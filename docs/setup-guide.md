# Invoice Scanner — Setup Guide

## Prerequisites
- Claude Code desktop app running on macOS
- Gmail MCP connected (baruch.halag@gmail.com)
- Google Drive MCP connected
- The CPA app "חדד לוי רואי חשבון" installed on this Mac

## First-Time Setup (do once)

### Step 1: Initialize the system
Open Claude Code and run:
```
/invoice-scanner setup
```
This creates the data files in Google Drive and registers the daily scheduled task.

### Step 2: Record the CPA app upload flow
```
/invoice-scanner teach
```
Have the CPA app open. Claude will watch you upload one invoice and record the steps.
This only needs to be done once (or after the app's UI changes significantly).

### Step 3: Add your known regular suppliers
For each supplier you regularly receive deductible invoices from:
```
/invoice-scanner add-supplier <supplier name>
```
Example: `/invoice-scanner add-supplier Partner Communications`

### Step 4: Run a test
```
/invoice-scanner run
```
This runs the full pipeline manually. Check your Gmail drafts for the digest.

### Step 5: Confirm the schedule
The system now runs automatically every day at 08:30 Israel time.
Check: `/invoice-scanner status`

---

## Daily Usage

### Reviewing pending invoices
When you receive a digest email mentioning pending invoices, open Claude and run:
```
/invoice-approve
```
You'll be shown each pending invoice one by one and can approve or reject.

### Checking system status
```
/invoice-scanner status
```

### Pausing the daily scan (e.g., while traveling)
```
/invoice-scanner pause
/invoice-scanner resume
```

---

## Understanding the Digest Email

Each morning you'll receive a Gmail **draft** (check Gmail Drafts folder) summarizing:
- ✅ Invoices uploaded automatically (from known approved suppliers)
- ⏳ Invoices pending your review (from unknown suppliers)
- ❌ Invoices skipped (from blocked suppliers)
- ⚠️ Failed uploads (will retry tomorrow)

The digest is created as a draft so you can review it before sending, or simply read it in the Drafts folder.

---

## Troubleshooting

**"Teaching session not completed" error**
Run `/invoice-scanner teach` to record the CPA app upload flow.

**Invoices not being found in Gmail**
The scanner looks for emails with attachments from the last 2 days containing invoice keywords.
Make sure your invoice emails arrive at baruch.halag@gmail.com.

**CPA upload failing**
The system retries up to 3 times. If it fails permanently, it will appear in the digest.
You may need to re-run the teaching session if the app's UI has changed: `/invoice-scanner teach`

**Supplier matched incorrectly**
Use `/invoice-scanner add-supplier` to add the correct entry, or run `/invoice-approve` to correct.
