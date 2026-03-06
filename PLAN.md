# TAM Metrics Executive Reporting System

## Context

The Yuno TAM (Technical Account Manager) team is perceived as "just a bug-fixing team" — but in reality they do onboarding, optimization, health checks, strategic advisory, and proactive merchant work. The goal is to change this narrative by building a reporting system that:

1. Categorizes TAM work into types (not just "bug fix")
2. Correlates TAM interventions with merchant business metrics (approval rates, transaction volume)
3. Generates two-layer reports: executive summary (C-level/VP) + detailed breakdown (team)
4. Delivers reports on-demand via CLI and weekly automated

## Architecture Decision: Skill-Only (No Python Agent)

All 4 data sources are covered by existing or easily-added MCPs. No Python agent needed.

| Source | MCP | What it contributes |
|--------|-----|---------------------|
| Jira | `mcp__plugin_yuno_atlassian` (already installed) | Tickets, work types, worklog, merchant context |
| Datadog | `mcp__plugin_yuno_datadog` (already installed) | Merchant metrics before/after intervention |
| Slack | `korotovsky/slack-mcp-server` (to add) | Channel threads, merchant conversations, async support context |
| Google Calendar | `nspady/google-calendar-mcp` (to add) | QBRs, onboarding calls, merchant meetings, time investment |

---

## Step 1: Add Missing MCPs

### Auth strategy: user signs in, no raw tokens

Both Slack and Google Calendar use an OAuth browser sign-in flow. Team members never manage bot tokens or GCP credentials — they just click "Authorize" once.

---

### 1a. Slack MCP — OAuth browser sign-in

**Package**: `korotovsky/slack-mcp-server` in OAuth mode (not bot token mode)

**One-time setup (Pedro only)**:
1. [api.slack.com/apps](https://api.slack.com/apps) → Create App → From scratch → name: `Claude TAM Report`
2. OAuth & Permissions → Redirect URLs → add `http://localhost:3000/callback`
3. Bot Token Scopes: `channels:history`, `channels:read`, `groups:history`, `search:messages`, `users:read`
4. Copy Client ID + Client Secret → store in 1Password

**Per-user setup**:
```bash
# Add to ~/.zshenv (values from 1Password)
export SLACK_CLIENT_ID=...
export SLACK_CLIENT_SECRET=...
```

First run opens a browser → user clicks "Allow" → token stored locally. No bot token needed.

**`.mcp.json` entry** (already configured):
```json
"slack": {
  "command": "npx",
  "args": ["-y", "@korotovsky/slack-mcp-server",
           "--client-id", "${SLACK_CLIENT_ID}",
           "--client-secret", "${SLACK_CLIENT_SECRET}"],
  "env": {
    "SLACK_CLIENT_ID": "${SLACK_CLIENT_ID}",
    "SLACK_CLIENT_SECRET": "${SLACK_CLIENT_SECRET}"
  }
}
```

> **Fallback**: If OAuth has issues (known Claude Code DCR bugs), use a shared `SLACK_BOT_TOKEN` from 1Password.

---

### 1b. Google Calendar MCP — OAuth browser sign-in

**Package**: `nspady/google-calendar-mcp`

**One-time setup (Pedro only)**:
1. [console.cloud.google.com](https://console.cloud.google.com) → new project → enable Google Calendar API
2. Credentials → Create OAuth 2.0 Client ID → Desktop app → download JSON → rename `gcal-client.json`
3. Share `gcal-client.json` via 1Password (this is the app identity, safe to share)

**Per-user setup**:
```bash
mkdir -p ~/.config/tam-metrics
cp gcal-client.json ~/.config/tam-metrics/gcal-client.json  # from 1Password
export GOOGLE_CREDENTIALS_PATH=~/.config/tam-metrics/gcal-client.json
```

First run opens a browser → user signs in with their Google account → personal token auto-saved. No GCP access needed by end users.

**`.mcp.json` entry** (already configured):
```json
"google-calendar": {
  "command": "npx",
  "args": ["-y", "@nspady/google-calendar-mcp"],
  "env": { "GOOGLE_CREDENTIALS_PATH": "${GOOGLE_CREDENTIALS_PATH}" }
}
```

---

## Step 2: Build the `yuno:tam-report` Skill

**File**: `/Users/pedro/.claude/plugins/cache/yuno-plugins/yuno/0.0.17/skills/tam-report/SKILL.md`

**Pattern**: Follows `datadog/SKILL.md` (bash + MCP hybrid) and `atlassian-jira/SKILL.md` (MCP usage).

### Skill frontmatter
```yaml
---
name: tam-report
description: "Generate TAM executive report. Triggers: TAM report, account manager
  metrics, merchant impact, tam metrics, what has tam been working on, tam team
  report, TAM impact, account management report"
allowed-tools: >
  mcp__plugin_yuno_atlassian__searchJiraIssuesUsingJql,
  mcp__plugin_yuno_atlassian__getJiraIssue,
  mcp__plugin_yuno_atlassian__createConfluencePage,
  mcp__plugin_yuno_atlassian__updateConfluencePage,
  mcp__plugin_yuno_datadog__query_metrics,
  mcp__plugin_yuno_datadog__get_monitors,
  mcp__slack__search_messages,
  mcp__slack__get_channel_history,
  mcp__google-calendar__list_events,
  Bash(source:*), Bash(date:*), Bash(jq:*), Read
---
```

### Skill behavior

**Invocation examples**:
- `/tam-report` → last 30 days, both layers
- `/tam-report --period 90d` → last quarter
- `/tam-report --tam pedro@yuno.com` → filter by TAM owner
- `/tam-report --merchant acme` → focus on one merchant
- `/tam-report --publish` → also push to Confluence

**Execution flow**:

```
STEP 1 — Fetch Jira tickets
  JQL: project = TAM AND updated >= -{period} ORDER BY updated DESC
  Fields: key, summary, assignee, status, labels, resolutiondate, description, comments

STEP 2 — Classify each ticket into WorkType
  Ask Claude inline (no API call needed — Claude IS the runtime):
  For each ticket → assign one of:
    INCIDENT_RESPONSE, BUG_FIX_ESCALATION,
    INTEGRATION_SUPPORT, OPTIMIZATION,
    HEALTH_CHECK, ONBOARDING,
    CONFIGURATION, EDUCATION

STEP 3 — Extract merchant context
  From ticket: check custom field "Merchant ID" or "Account"
  Fallback: regex match merchant name from title/description

STEP 4 — Enrich from Slack
  For each merchant found: search Slack for mentions in TAM channels
  Pull thread context ±7 days around ticket dates
  Adds: "what conversations happened around this issue"

STEP 5 — Enrich from Google Calendar
  For each TAM owner: list events in the period
  Filter: events with merchant name in title/attendees
  Categorizes: QBRs, onboarding calls, health checks

STEP 6 — Fetch Datadog metrics (for resolved tickets only)
  For each resolved ticket with known merchant:
    Query approval rate / error rate in two windows:
      baseline: [resolution_date - 30d, resolution_date - 7d]
      post:     [resolution_date + 7d,  resolution_date + 37d]
    Compute delta, assign confidence level:
      HIGH   → delta > 2%  and post_days >= 14
      MEDIUM → delta > 0.5% and post_days >= 7
      LOW    → delta exists but thresholds not met
      NONE   → no data or too recent

STEP 7 — Generate report (two layers)
```

---

## Report Format

### Executive Summary (C-level / VP layer)

```markdown
## TAM Impact — [Period]

**Portfolio**: N merchants · X interventions · Y% proactive work

### Business Impact (High Confidence)
| Merchant | TAM Activity | Approval Rate Change | Est. Txns Recovered |
|----------|-------------|---------------------|---------------------|
| Acme Corp | Routing optimization | +3.2% | ~14,000 |
| Beta Inc  | Webhook retry fix   | +1.8% | ~6,400  |

### Work Distribution (TAM != bug fixes)
| Type | Count | % |
|------|-------|---|
| Onboarding | 8 | 25% |
| Optimization | 7 | 22% |
| Integration Support | 6 | 19% |
| Health Check | 5 | 16% |
| Incident Response | 4 | 12% |
| Bug Fix Escalation | 2 | 6% |

### Merchants Needing Attention
- [Merchant X] — approval rate down 1.5%, open ticket >7d
- [Merchant Y] — no TAM contact in 45d
```

### Detailed TAM Team Breakdown

```markdown
## TAM Operations — [Period]

### By Owner
| TAM | Tickets | Proactive % | Merchants | Meetings | Highlight |

### All Interventions
| Key | Merchant | Category | Status | Resolved | Signal | Confidence |

### Slack Activity Summary
[Threads per merchant, key conversations]

### Calendar Summary
[QBRs held, onboarding calls, hours invested per merchant]

### Metric Attribution Notes
[Confidence methodology, exclusions, caveats]
```

---

## Work Type Taxonomy

```
INCIDENT_RESPONSE     → Reactive: outage or degradation response
BUG_FIX_ESCALATION   → Reactive: confirmed bug escalated to engineering
INTEGRATION_SUPPORT  → Proactive: new merchant going live
OPTIMIZATION         → Proactive: improving existing integration
HEALTH_CHECK         → Proactive: QBR, periodic merchant review
ONBOARDING           → Proactive: new merchant activated end-to-end
CONFIGURATION        → Proactive: routing rules, payment method setup
EDUCATION            → Proactive: documentation, training, best practices
```

**Key narrative**: Reactive (INCIDENT + BUG) should ideally be <30% of total. The rest proves TAM drives proactive value.

---

## Build Order

1. **Confirm Jira project key** — run quick JQL via Jira MCP to verify tickets exist
2. **Add Slack MCP** — install `korotovsky/slack-mcp-server`, get bot token, add to `.mcp.json`
3. **Add Google Calendar MCP** — install `nspady/google-calendar-mcp`, OAuth flow, add to `.mcp.json`
4. **Create `tam-report/SKILL.md`** — write skill following `datadog/SKILL.md` patterns
5. **Test Phase 1** — run `/tam-report` with just Jira + WorkType classification
6. **Test Phase 2** — add Datadog impact correlation for resolved tickets
7. **Test Phase 3** — validate Slack + Calendar enrichment
8. **Tune report format** — iterate with TAM team on categories and layout
9. **Add `--publish` flag** — push executive summary to Confluence via Atlassian MCP
10. **Weekly automation** — add cron entry: `claude -p "/tam-report --publish"` every Monday 9am

---

## Verification

- `/tam-report` shows last 30 days of TAM tickets with work type distribution
- WorkType breakdown shows >40% proactive (if data is correct)
- At least one HIGH/MEDIUM confidence Datadog attribution per report
- Executive summary fits in one Slack message / one Confluence page
- `--publish` creates or updates the Confluence page without duplicating

---

## Key Reference Files (for implementation)

| File | Purpose |
|------|---------|
| `.../skills/datadog/SKILL.md` | Bash helper function pattern + credential loading |
| `.../skills/atlassian-jira/SKILL.md` | MCP tool usage pattern in skills |
| `.../skills/redash/SKILL.md` | Complex multi-step query pattern |
| `.../docs/DEVELOPER_GUIDE.md` | Canonical skill creation guide |

## MCP References

| MCP | Source |
|-----|--------|
| Slack | https://github.com/korotovsky/slack-mcp-server |
| Google Calendar | https://github.com/nspady/google-calendar-mcp |
| Google Official | https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services |
