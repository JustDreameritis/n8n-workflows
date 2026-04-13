# n8n Workflow Examples

Three production-ready n8n workflow templates covering common automation patterns.

## Workflows

### 1. Webhook → Process → Telegram Alert
**File:** `1-webhook-telegram-alert.json`

Receives incoming POST data via webhook, normalizes the payload, filters by severity level, and sends formatted alerts to Telegram. Includes a response node that confirms receipt to the caller.

**Nodes:** Webhook → Code (process) → Filter (skip info) → Telegram → Respond

**Use cases:** deployment notifications, monitoring alerts, form submissions, third-party event hooks.

```bash
# Test it
curl -X POST http://localhost:5678/webhook/incoming-data \
  -H "Content-Type: application/json" \
  -d '{"event": "deploy", "severity": "high", "message": "v2.1 deployed"}'
```

### 2. RSS Monitor → Keyword Filter → Google Sheets
**File:** `2-rss-filter-sheets.json`

Polls an RSS feed every 15 minutes, filters items by a configurable keyword list, and appends matches to a Google Sheet with metadata (matched keywords, source, timestamp).

**Nodes:** Schedule (15min) → RSS Read → Code (keyword filter) → Filter (has matches) → Google Sheets

**Use cases:** content monitoring, competitive intelligence, job board scanning, news aggregation.

### 3. API Call → Transform → POST to Another API
**File:** `3-api-transform-post.json`

Fetches data from a source API (GitHub issues in the example), transforms the response into a clean normalized schema, filters by criteria, and forwards to a downstream API endpoint.

**Nodes:** Schedule (1hr) → HTTP GET → Code (transform) → Filter (unassigned & fresh) → HTTP POST

**Use cases:** API synchronization, data pipeline ETL, cross-platform integrations, webhook relay.

## Import

1. Open your n8n instance
2. Click **...** menu → **Import from file**
3. Select any JSON file from this repo
4. Configure credentials (Telegram, Google Sheets) as needed

## License

MIT
