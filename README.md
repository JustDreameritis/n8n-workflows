# n8n Workflow Templates

Production-ready n8n workflow templates for common automation patterns. Import directly into your n8n instance.

---

## Workflows

### 1. Webhook → Telegram Alert
**File:** `1-webhook-telegram-alert.json`

Receive webhook events and forward as formatted Telegram messages with severity-based filtering. Processes and normalizes incoming POST data, skips `info`-level events, and sends HTML-formatted alerts for `medium` and above.

**Nodes:** Webhook → Code (normalize + emoji) → Filter (skip info) → Telegram → Respond 200

**Use cases:** deployment notifications, monitoring alerts, form submissions, third-party webhooks.

```bash
curl -X POST http://localhost:5678/webhook/incoming-data \
  -H "Content-Type: application/json" \
  -d '{"event": "deploy", "severity": "high", "message": "v2.1 deployed to prod"}'
```

---

### 2. RSS Feed → Google Sheets
**File:** `2-rss-filter-sheets.json`

Monitor RSS feeds on a 15-minute schedule, filter items by configurable keyword list, normalize schema, and append matching items to Google Sheets with deduplication.

**Nodes:** Schedule (15 min) → RSS Read → Code (keyword filter) → Filter (has matches) → Google Sheets

**Use cases:** content monitoring, competitive intelligence, job board scanning, news aggregation.

---

### 3. API Transform → POST
**File:** `3-api-transform-post.json`

Fetch data from an external API hourly, transform and normalize the response, filter by configurable criteria, and forward processed items to a downstream service. Includes retry logic.

**Nodes:** Schedule (1 hr) → HTTP GET → Code (transform + normalize) → Filter → HTTP POST

**Use cases:** API sync pipelines, ETL automation, cross-platform integrations, webhook relay.

---

### 4. Lead Capture → CRM
**File:** `4-lead-capture-crm.json`

Full lead intake pipeline with validation, deduplication, CRM entry creation, and multi-channel notification.

**Flow:**
1. Webhook receives form submission (name, email, company, message)
2. Code node validates email format and trims all fields
3. Airtable API checks for existing record by email (dedup)
4. IF new lead: creates Airtable record → sends confirmation email via SMTP → notifies sales team on Telegram → responds 200
5. IF duplicate: responds 409 with friendly message
6. Error path: dead-letter queue captures failures

**Nodes:** Webhook → Validate (Code) → Dedup Check (HTTP) → IF → CRM Create (HTTP) → Email → Telegram → Respond · Error → Dead Letter

**Canvas diagram:** [screenshots/4-lead-capture-crm.html](screenshots/4-lead-capture-crm.html)

**Credentials needed:** Airtable API token, SMTP, Telegram Bot

---

### 5. Email Attachment → Google Sheets Parser
**File:** `5-email-attachment-sheets.json`

Watches an inbox via IMAP, extracts CSV or Excel attachments, parses every row, validates and normalizes column names, and appends to a Google Sheet. Sends a summary notification on completion.

**Flow:**
1. IMAP trigger fires on each new email
2. IF node checks for attachments — skips emails without them
3. Code node finds the first `.csv`/`.xlsx`/`.xls` attachment and extracts binary
4. SpreadsheetFile node parses rows from the binary
5. Code node normalizes keys (lowercase, underscores) and adds import metadata
6. Google Sheets appends all rows in a single batch
7. Code node aggregates row count and source metadata
8. Telegram sends summary (file name, sender, row count, sheet link)
9. Error path catches bad formats and alerts via Telegram

**Nodes:** IMAP → IF (has attachment) → Extract (Code) → Parse (SpreadsheetFile) → Transform (Code) → Sheets → Summary (Code) → Telegram · Error → DLQ → Telegram alert

**Canvas diagram:** [screenshots/5-email-attachment-sheets.html](screenshots/5-email-attachment-sheets.html)

**Credentials needed:** IMAP account, Google Sheets OAuth2, Telegram Bot

---

### 6. Multi-Channel Alert Router
**File:** `6-multichannel-alert-router.json`

Intelligent alert dispatcher with severity-based routing, in-memory rate limiting, multi-channel fan-out, and a full audit trail in Google Sheets.

**Flow:**
1. Webhook receives alert payload (`source`, `severity`, `message`, `metadata`)
2. Code node normalizes payload and applies rate limiter (5 alerts/source/minute)
3. IF rate limited: responds 429
4. Switch routes by severity:
   - **critical** → Telegram + Slack (#critical) + Email → Merge → Audit log
   - **warning** → Telegram + Slack (#alerts) → Merge → Audit log
   - **info** → Slack (#general) → Audit log
5. All paths merge before Google Sheets audit append
6. Responds with `alertId` and routed severity

**Nodes:** Webhook → Normalize+RateLimit (Code) → IF (rate limit) → Switch (severity) → [Telegram | Slack | Email] × lanes → Merge → Google Sheets audit → Respond

**Canvas diagram:** [screenshots/6-multichannel-alert-router.html](screenshots/6-multichannel-alert-router.html)

**Credentials needed:** Telegram Bot, Slack OAuth2, SMTP, Google Sheets OAuth2

---

## How to Import

1. Open your n8n instance
2. Go to **Workflows** → **...** menu → **Import from File**
3. Select any `.json` file from this repo
4. Open each node and update credentials in the node settings panel
5. Set required environment variables (see below)
6. Click **Activate** to go live

---

## Environment Variables

Configure these in your n8n instance settings or `.env` file:

| Variable | Used By | Description |
|----------|---------|-------------|
| `TELEGRAM_CHAT_ID` | 1, 4, 5, 6 | Telegram chat/group ID for alerts |
| `TELEGRAM_SALES_CHAT_ID` | 4 | Separate channel for sales notifications |
| `TELEGRAM_ALERT_CHAT_ID` | 6 | Ops alert channel |
| `AIRTABLE_BASE_ID` | 4 | Airtable base ID (find in API docs) |
| `SHEETS_DOC_ID` | 5 | Google Sheets document ID |
| `AUDIT_SHEETS_DOC_ID` | 6 | Separate sheet for alert audit log |
| `SMTP_FROM` | 4, 6 | From address for outbound email |
| `OPS_EMAIL` | 6 | Ops team email for critical alerts |
| `IMAP_HOST` | 5 | IMAP server hostname |
| `IMAP_USER` | 5 | IMAP username / email address |
| `IMAP_PASSWORD` | 5 | IMAP password or app-specific password |
| `SLACK_CRITICAL_CHANNEL` | 6 | Slack channel ID for critical alerts |
| `SLACK_ALERTS_CHANNEL` | 6 | Slack channel ID for warning alerts |
| `SLACK_INFO_CHANNEL` | 6 | Slack channel ID for info alerts |

---

## Customization

All workflows are designed as starting points. Common modifications:

- **Swap notification channels** — replace Telegram nodes with Discord webhooks, or Slack with Teams
- **Change the CRM** — swap the Airtable HTTP Request nodes for HubSpot, Pipedrive, or Salesforce API calls
- **Add data sources** — duplicate trigger nodes and use a Merge node to combine outputs
- **Adjust rate limits** — edit the `MAX_PER_SOURCE` and `WINDOW_MS` constants in the Code node of workflow 6
- **Custom transformations** — the Code/Function nodes are plain JavaScript — edit them directly in n8n
- **Extend error handling** — all error paths are wired, add additional notification nodes as needed

---

## Architecture Pattern

All workflows follow a consistent structure:

```
Trigger → Validate/Normalize → Business Logic → Output(s) → Respond
                                     ↓
                               Error Handler → Dead Letter Queue
```

- **Input validation** at the entry node
- **Idempotency guards** (dedup checks, rate limiters) before side-effects
- **Error handling** on every branch with dead-letter capture
- **Audit trail** for anything that modifies external state

---

## Tech Stack

- [n8n](https://n8n.io) — open-source workflow automation platform
- Compatible with n8n Cloud and self-hosted instances
- Tested on **n8n v1.30+**
- Node types used: `webhook`, `emailReadImap`, `code`, `if`, `switch`, `filter`, `merge`, `httpRequest`, `telegram`, `slack`, `emailSend`, `googleSheets`, `spreadsheetFile`, `respondToWebhook`, `noOp`

---

## License

MIT — use these templates freely in your projects, commercial or otherwise.

---

*Built by [JustDreameritis](https://github.com/JustDreameritis) — Automation & Integration Developer*
