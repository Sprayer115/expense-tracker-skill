---
name: expense-tracker
description: Log an expense to your Google Sheet by describing it naturally.
require-secret: true
homepage: https://github.com/Sprayer115/expense-tracker-skill
---

# Expense Tracker

Log expenses to your personal Google Sheet just by describing them. The skill extracts the relevant details and appends a new row automatically.

## Secret

The secret is your **Google Apps Script Web App URL**. See the setup section below for how to get it.

## Instructions

You are an expense logging assistant. When the user describes an expense (or asks to log one), extract the following fields:

- **date**: The expense date in YYYY-MM-DD format. Default to today's date if not mentioned.
- **amount**: A numeric value (e.g. 12.50). Ask if unclear.
- **category**: One of: Food, Transport, Shopping, Health, Entertainment, Bills, Other. Infer from context.
- **description**: A short description of what was purchased/spent on.
- **payment_method**: One of: Cash, Card, PayPal, Bank Transfer, Other. Default to Card if not mentioned.

Before submitting, briefly confirm the details with the user in a single line (e.g. "Logging: 2026-04-08 | €12.50 | Food | Groceries at Rewe | Card — correct?").

Once confirmed (or if the user says to just log it), call `run_js` with:
- script_name: `index.html`
- data: a JSON string with the fields above, e.g.:
  `{"date":"2026-04-08","amount":12.50,"category":"Food","description":"Groceries at Rewe","payment_method":"Card"}`

After the tool returns:
- If `result` is present: tell the user the expense was logged successfully.
- If `error` is present: tell the user something went wrong and show the error message.

## Setup

You need to do a one-time setup to connect this skill to your Google Sheet.

### Step 1 — Prepare your Google Sheet

Open your Google Sheet and make sure the first row has these headers (in this order):

| Date | Amount | Category | Description | Payment Method | Logged At |

### Step 2 — Deploy the Apps Script

1. In your Google Sheet: **Extensions → Apps Script**
2. Delete any existing code and paste in the following:

```javascript
// Health check — open the Web App URL in a browser to verify it's deployed
function doGet(e) {
  return ContentService
    .createTextOutput(JSON.stringify({ status: "Expense Tracker is running" }))
    .setMimeType(ContentService.MimeType.JSON);
}

function doPost(e) {
  try {
    var raw = e.postData.contents;
    var data = JSON.parse(raw);
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    sheet.appendRow([
      data.date,
      data.amount,
      data.category,
      data.description,
      data.payment_method,
      new Date().toISOString()
    ]);
    return ContentService
      .createTextOutput(JSON.stringify({ result: "ok" }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ error: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

3. Click **Deploy → New deployment**
4. Type: **Web app**
5. Execute as: **Me**
6. Who has access: **Anyone**
7. Click **Deploy** and copy the **Web App URL**

### Step 3 — Set the secret in Google AI Edge Gallery

When prompted for the secret in the app, paste the Web App URL you copied above.

### Testing the webhook

**Step 1 — Browser health check**

Open your Web App URL directly in a browser. You should see:
```json
{"status":"Expense Tracker is running"}
```
If you see a Google login page or 403, re-check the deployment settings (Access must be **Anyone**).

**Step 2 — POST test with curl**

```bash
curl -s -L 'https://script.google.com/macros/s/AKfycbycMIN3geTEzuI2nPlkHZQBOuSA4nvtbehUTSqtapy6Hezu0YcIyir1ydyjRbU4rHUz/exec' -H 'Content-Type: application/json' -d '{"date":"2026-04-08","amount":9.99,"category":"Food","description":"Test coffee","payment_method":"Card"}'
```

Expected response: `{"result":"ok"}`

> **Important:** Do **not** add `-X POST` to the curl command. Apps Script URLs return a 302 redirect; curl handles it correctly when using `-d` alone (POST on first request, GET on redirect). Adding `-X POST` forces POST on the redirect too — but without a body — causing a `411 Length Required` error.
