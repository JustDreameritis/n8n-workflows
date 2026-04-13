# n8n Workflow Automation

Production-ready n8n workflow patterns for real-time data processing, monitoring, and multi-system integration.

## Architecture

```mermaid
graph TB
    subgraph ENGINE["n8n AUTOMATION ENGINE"]
        N8N["n8n Core<br/><i>Event-driven execution</i><br/><i>Credential management</i>"]
    end

    subgraph WF1["WORKFLOW 1 — Webhook Alert"]
        WH["Webhook<br/>POST /incoming-data"]
        PROC["Process & Validate<br/><i>Normalize payload</i><br/><i>Add severity emoji</i>"]
        FILT1["Filter<br/><i>Skip info level</i>"]
        TG["Telegram Alert<br/><i>HTML formatted</i>"]
        RESP["Respond 200 OK"]
    end

    subgraph WF2["WORKFLOW 2 — RSS Monitor"]
        SCHED1["Schedule<br/><i>Every 15 min</i>"]
        RSS["RSS Read<br/><i>Hacker News</i>"]
        KW["Keyword Filter<br/><i>python, typescript,</i><br/><i>n8n, claude, llm</i>"]
        SHEETS["Google Sheets<br/><i>Append row</i>"]
    end

    subgraph WF3["WORKFLOW 3 — API Transform"]
        SCHED2["Schedule<br/><i>Every 1 hour</i>"]
        GET["HTTP GET<br/><i>GitHub Issues API</i>"]
        XFORM["Transform<br/><i>Extract bounty, age</i><br/><i>Normalize schema</i>"]
        FILT3["Filter<br/><i>Unassigned & fresh</i>"]
        POST["HTTP POST<br/><i>Downstream API</i>"]
    end

    subgraph SHARED["SHARED INFRASTRUCTURE"]
        ERR["Error Handler<br/><i>Retry + dead letter</i>"]
        LOG["Execution Log<br/><i>Success/failure tracking</i>"]
        ALERT["Admin Alert<br/><i>Telegram on failure</i>"]
    end

    N8N --> WH
    N8N --> SCHED1
    N8N --> SCHED2

    WH --> PROC --> FILT1 --> TG
    PROC --> RESP

    SCHED1 --> RSS --> KW --> SHEETS

    SCHED2 --> GET --> XFORM --> FILT3 --> POST

    WF1 -.-> ERR
    WF2 -.-> ERR
    WF3 -.-> ERR
    ERR --> LOG
    ERR --> ALERT

    style ENGINE fill:#1a1a2e,stroke:#4a9eff,stroke-width:2px,color:#e0e0e0
    style WF1 fill:#0f1628,stroke:#00d4aa,stroke-width:2px,color:#ccc
    style WF2 fill:#0f1628,stroke:#ffd93d,stroke-width:2px,color:#ccc
    style WF3 fill:#0f1628,stroke:#ff9f43,stroke-width:2px,color:#ccc
    style SHARED fill:#0f1628,stroke:#ff6b6b,stroke-width:2px,color:#ccc
```

## Workflows

### 1. Webhook → Process → Telegram Alert
**File:** `1-webhook-telegram-alert.json`

Receives incoming POST data, validates payload, routes by severity level, sends formatted Telegram alerts for high-priority events. Includes error handling and response confirmation.

**Nodes:** Webhook → Code (process) → Filter (skip info) → Telegram → Respond

**Use cases:** deployment notifications, monitoring alerts, form submissions, third-party event hooks.

```bash
# Test it
curl -X POST http://localhost:5678/webhook/incoming-data \
  -H "Content-Type: application/json" \
  -d '{"event": "deploy", "severity": "high", "message": "v2.1 deployed"}'
```

### 2. RSS Monitor → Filter → Google Sheets
**File:** `2-rss-filter-sheets.json`

Polls RSS feeds on schedule, filters items by configurable keyword matching, normalizes data structure, appends matching items to Google Sheets with timestamps. Deduplication built in.

**Nodes:** Schedule (15min) → RSS Read → Code (keyword filter) → Filter (has matches) → Google Sheets

**Use cases:** content monitoring, competitive intelligence, job board scanning, news aggregation.

### 3. API → Transform → POST
**File:** `3-api-transform-post.json`

Fetches data from external API on schedule, transforms and normalizes response, filters by criteria, forwards processed data to destination API. Includes retry logic and error logging.

**Nodes:** Schedule (1hr) → HTTP GET → Code (transform) → Filter (unassigned & fresh) → HTTP POST

**Use cases:** API synchronization, data pipeline ETL, cross-platform integrations, webhook relay.

## Architecture Pattern

All workflows follow the same pattern:
- Input validation at entry point
- Data transformation in middle layer
- Error handling on every path
- Dead letter queue for failed items
- Admin alerting on failures

## Import

1. Open your n8n instance
2. Click **...** menu → **Import from file**
3. Select any JSON file from this repo
4. Configure credentials (Telegram, Google Sheets) as needed

## License

MIT
