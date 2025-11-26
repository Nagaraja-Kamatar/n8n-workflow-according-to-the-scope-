<img width="856" height="535" alt="n8n" src="https://github.com/user-attachments/assets/d6ef8be7-f06e-4037-88b2-1503d1e70c52" />


<img width="1431" height="703" alt="reteal ai" src="https://github.com/user-attachments/assets/12ece747-9e65-4351-89a7-70363b476f09" />


<img width="468" height="721" alt="Screenshot 2025-11-26 at 5 43 04 PM" src="https://github.com/user-attachments/assets/f071bd32-bab0-49a0-be9e-f98736a0f68a" />



# n8n-ADRVA Starter

This repository is a customized version of the **n8n-nodes-starter** tailored to the **Automated Debt Recovery Voice Agent (ADRVA)** project. ADRVA uses n8n to stitch together Google Sheets (debtors), Google Drive (product docs), Retell AI / OpenAI (Cindy agent), and Twilio (telephony) to place polite, compliant voice calls, answer debtor questions, schedule callbacks, and log everything for managers.

---

## Project Overview

**Project name:** Automated Debt Recovery Voice Agent (ADRVA) — n8n Beta

**Purpose:** Build an n8n integration and workflow that:

* Reads debtors from Google Sheets
* Reads product documentation from Google Drive
* Calls debtors via Twilio
* Uses Retell AI / OpenAI for natural voice conversation (Cindy persona)
* Parses intents (payment, callback, transfer, info)
* Logs outcomes to Google Sheets and generates HTML manager reports

**Core services used:** Google Sheets, Google Drive, Twilio, OpenAI (gpt-4o-mini, tts-1, whisper-1), n8n.

---

## Repository Contents (customized)

* `nodes/` - example nodes and any ADRVA-specific nodes (e.g., GoogleDriveParser, RetellAIConnector)
* `credentials/` - sample credential definitions for Google, Twilio, and OpenAI
* `workflows/` - example n8n JSON workflows (import-ready): `adrva-main-workflow.json`
* `docs/` - project docs: README, deployment guides, compliance checklist
* `scripts/` - helper scripts (callback parser, PDF extractor helpers)

---

## Quick Start (ADRVA-specific)

> Prerequisites: Node.js v22+, npm, git, an n8n development environment (see below).

1. Clone this repository created from the n8n starter template.

```bash
git clone https://github.com/Nagaraja-Kamatar/n8n-workflow-according-to-the-scope-.git
cd n8n-adrva-starter
npm install
```

2. Prepare environment variables (create a `.env` file). Example `.env`:

```
N8N_HOST=http://localhost
N8N_PORT=5678
GOOGLE_OAUTH_CLIENT_ID=xxx
GOOGLE_OAUTH_CLIENT_SECRET=xxx
TWILIO_ACCOUNT_SID=xxx
TWILIO_AUTH_TOKEN=xxx
TWILIO_FROM_NUMBER=+1234567890
OPENAI_API_KEY=sk-xxx
COMPANY_NAME="AI Workshop"
COMPANY_PHONE=+11234567890
```

3. Run dev server (hot reload):

```bash
npm run dev
```

Open n8n at `http://localhost:5678` and confirm custom nodes are visible.

---

## ADRVA Workflow (high-level)

This repository contains `workflows/adrva-main-workflow.json` — a ready-to-import n8n workflow that implements the following:

1. **Trigger**: Cron / Manual / Webhook to start calling a batch of debtors.
2. **Google Sheets - Read Debtors**: Fetch rows from `Debtors` sheet (columns: id, name, phone, amount, product, purchase_date, product_doc_id, email, status, last_call_date, callback_date, notes).
3. **Google Drive - Get Product Doc**: If `product_doc_id` exists, download file and extract text via `nodes/scripts/pdf-extract` or 3rd-party OCR.
4. **Retell AI / OpenAI - Conversation**: Send debtor context + product doc snippet to LLM (system prompt = Cindy persona). Use `gpt-4o-mini` for intent; use `tts-1` for voice (or generate audio asset and pass to Twilio).
5. **Twilio - Make Call**: Place outbound call with TwiML; Twilio webhooks map to n8n webhook nodes to handle Gather / speech transcriptions.
6. **Function - Intent Parser**: Parse structured JSON from the LLM (intent, callback_time, payment_link, transfer, reply_text) and branch logic.
7. **Google Sheets - Log Call**: Append to `Logs` sheet with fields (timestamp, call_id, debtor_row_id, name, phone, amount, product, outcome, callback_date, notes, recording_url).
8. **Callback Scheduler**: If callback requested, either write to `Callbacks` sheet and let Cron pick it up or create a `Wait` node for low-volume setups.
9. **HTML Report**: Build a daily manager HTML report and save to Google Drive.

---

## Google Sheets Templates

Included in `workflows/templates/`:

* `Debtors.csv` (id,name,phone,amount,product,purchase_date,product_doc_id,email,status,last_call_date,callback_date,notes)
* `Logs.csv` (timestamp,call_id,debtor_row_id,name,phone,amount,product,outcome,callback_date,notes,recording_url)
* `Callbacks.csv` (debtor_row_id,callback_time,timezone,status)

Import these to your Google Drive and set the Sheet IDs in the workflow credential config.

---

## Node Details (custom helpers)

* **nodes/RetellAI/**

  * A connector node that wraps calls to OpenAI / Retell API, handles structured JSON response parsing, and optionally generates audio files with `tts-1`.

* **nodes/PDFExtractor/**

  * Extracts text from PDF binary blobs using an internal parser or an external OCR service (Google Vision / Textract integration). Returns `short_product_description` and `full_product_text`.

* **nodes/CallbackParser/**

  * Uses `chrono-node` to parse phrases like "in 3 days" or "tomorrow morning" into ISO8601 datetimes in the debtor timezone.

See each node's README under `nodes/*` for usage and config.

---

## Credentials Setup (exact names)

Create credentials in n8n with these names (the workflow references them by name):

* `googleSheetsOAuth` - Google Sheets (OAuth2 client id/secret) — scope: read/write to Drive & Sheets
* `googleDriveOAuth` - Google Drive (OAuth2 client id/secret)
* `twilioApi` - Twilio (Account SID + Auth Token + From number)
* `openaiApi` - OpenAI / Retell (API Key)

Set these credentials in n8n > Credentials before importing workflows.

---

## Retell AI / LLM System Prompt (Cindy persona)

Place this prompt in the RetellAI node as the system message (replace `{{company}}` variables programmatically):

```
You are Cindy, a professional and empathetic voice assistant calling from {{COMPANY_NAME}}. Use the debtor context and any product_doc_text to answer questions concisely and politely. If user requests a callback, output JSON with intent: 'callback' and callback_time in ISO8601. If user requests transfer, output intent: 'transfer'. If user asks about payment and requests a link, output intent: 'payment' and include payment_link. Always return a short reply_text suitable for TTS and a notes field for logs.
```

---

## Workflow Import & Testing

1. Import `workflows/adrva-main-workflow.json` into n8n (Workflows > Import).
2. Set the workflow credentials to the names above.
3. Do a dry run with a test `Debtors` row (use a personal number).
4. Monitor the `Logs` sheet for appended rows and inspect generated HTML reports in Drive.

---

## Production Notes & Compliance

* **PCI**: Do NOT collect full card numbers via voice unless you have a PCI-compliant process. Prefer secure payment links or transfer to a human PCI endpoint.
* **DNC**: Respect Do-Not-Call lists and local call-time restrictions. The workflow includes a check to skip numbers that match a `DNC` list sheet.
* **Consent**: Include consent language where required and announce call recording if enabled.
* **Scalability**: Replace long-running `Wait` nodes with a Cron + Callbacks sheet approach for high volume.

---

## CI / Linting / Tests

Use the provided scripts from the starter for dev:

* `npm run dev` — start n8n with local nodes
* `npm run lint` — lint the node code
* `npm run build` — build for production

Add unit tests for node helpers (CallbackParser, PDFExtractor) if you customize them.

---

## Publishing your node (optional)

Follow n8n guidelines to publish your community node package if you want others to reuse it. Ensure:

* MIT license
* No sensitive data in repo
* Follow the `n8n.nodes` registration convention

---

## Support & Next Steps

If you want, I can now:

* Generate the **ready-to-import `workflows/adrva-main-workflow.json`** (complete with placeholders for your Sheet IDs, Twilio number, and OpenAI key).
* Produce a **step-by-step node config guide** (exact fields to paste into each n8n node).
* Create a **short demo video script** to show the workflow end-to-end.

Tell me which of these you'd like next and I will generate it immediately.


Have suggestions for improving this starter? [Open an issue](https://github.com/n8n-io/n8n-nodes-starter/issues) or submit a pull request!

## License

[MIT](https://github.com/n8n-io/n8n-nodes-starter/blob/master/LICENSE.md)
