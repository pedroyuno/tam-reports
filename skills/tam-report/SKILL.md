---
name: tam-report
description: |
  Generate TAM executive report showing what the implementation team has been working on.
  Proves TAM work goes beyond bug fixes by categorizing work types and correlating with merchant business impact.
  Triggers: "TAM report", "account manager metrics", "merchant impact", "tam metrics",
  "what has tam been working on", "tam team report", "TAM impact", "implementation report",
  "IMP2 report", "what has the implementation team done"
allowed-tools: >
  mcp__plugin_yuno_atlassian__getAccessibleAtlassianResources,
  mcp__plugin_yuno_atlassian__searchJiraIssuesUsingJql,
  mcp__plugin_yuno_atlassian__getJiraIssue,
  mcp__plugin_yuno_atlassian__createConfluencePage,
  mcp__plugin_yuno_atlassian__updateConfluencePage,
  mcp__plugin_yuno_atlassian__searchConfluenceUsingCql,
  mcp__plugin_yuno_datadog__query_metrics,
  mcp__plugin_yuno_datadog__get_monitors,
  Bash(source:*), Bash(date:*), Bash(jq:*), Bash(curl:*), Read
---

# TAM / Implementation Team Report Skill

Generate executive-quality reports for the Yuno TAM (Technical Account Manager) team,
showing the full scope of their work and its business impact on merchants.

## Constants

```
CLOUD_ID=f97a69c7-4a91-4f0f-b71a-13457bb62267
JIRA_PROJECT=IMP2
JIRA_BASE_URL=https://yunopayments.atlassian.net
```

## Invocation

The user may call this skill with optional flags:

```
/tam-report                          → last 30 days, both report layers, current user
/tam-report --period 90d             → last quarter
/tam-report --period 7d              → last week
/tam-report --tam pedro              → filter by TAM name (partial match on assignee)
/tam-report --merchant acme          → focus on a specific merchant
/tam-report --publish                → also create/update a Confluence page
```

Parse any flags from the user's message before starting.

## Execution Flow

### STEP 0 — Check connections

```bash
source ~/.zshenv
MISSING=""
[[ -z "$DD_API_KEY" ]]              && MISSING="$MISSING\n  - Datadog (run /yuno-claude-plugin:login datadog)"
[[ -z "$SLACK_CLIENT_ID" && -z "$SLACK_BOT_TOKEN" ]] && MISSING="$MISSING\n  - Slack (run /tam-login)"
[[ -z "$GOOGLE_CREDENTIALS_PATH" ]] && MISSING="$MISSING\n  - Google Calendar (run /tam-login)"

if [[ -n "$MISSING" ]]; then
  echo "Note: some data sources not configured — report will use available sources only:"
  echo -e "$MISSING"
  echo ""
  echo "Run /tam-login to set them up."
fi
```

Proceed regardless. Slack and Calendar are enrichment — the report works without them.

### STEP 1 — Fetch Jira tickets from IMP2

Use `searchJiraIssuesUsingJql` with cloudId `f97a69c7-4a91-4f0f-b71a-13457bb62267`.

Build JQL based on flags:
- Default (no flags): `project = IMP2 AND updated >= -30d ORDER BY updated DESC`
- With `--period 90d`: `project = IMP2 AND updated >= -90d ORDER BY updated DESC`
- With `--tam [name]`: add `AND assignee in membersOf("implementation") AND assignee ~ "[name]"`
- With `--merchant [name]`: add `AND summary ~ "[name]"`

Request these fields: `summary, assignee, status, issuetype, resolutiondate, created, updated, labels, description`

Fetch up to 50 tickets. If more than 50, note the count and fetch the most recent.

### STEP 2 — Map to WorkType

For each ticket, assign a WorkType based on **issue type** and **status**. Use this mapping as the primary guide, but use the ticket summary/description to refine when ambiguous:

| Jira Issue Type | Jira Status contains | WorkType |
|----------------|---------------------|----------|
| New Merchant | any | ONBOARDING |
| Upsale | any | INTEGRATION_SUPPORT |
| any | "Kick-off" or "kickoff" | ONBOARDING |
| any | "Sandbox" or "certification" | INTEGRATION_SUPPORT |
| any | "LIVE - EXTENDED CARE" | HEALTH_CHECK |
| any | "Done" or "Live" | ONBOARDING (completed) |
| Bug / Incident | any | BUG_FIX_ESCALATION |
| Support / Task | any | Read summary to decide |

**WorkType taxonomy** (use for all tickets not clearly mapped above):
- `ONBOARDING` — New merchant integration from scratch
- `INTEGRATION_SUPPORT` — Adding new payment method/provider to existing merchant
- `OPTIMIZATION` — Improving performance, routing, or approval rates for live merchant
- `HEALTH_CHECK` — Periodic review, QBR, extended care monitoring
- `CONFIGURATION` — Routing rules, payment method setup, configuration changes
- `BUG_FIX_ESCALATION` — Confirmed bug escalated to engineering
- `INCIDENT_RESPONSE` — Active outage or degradation
- `EDUCATION` — Documentation, training, best practices

**Key insight**: In IMP2, most work is ONBOARDING and INTEGRATION_SUPPORT — this is the evidence that TAM does far more than bug fixes.

### STEP 3 — Extract merchant names

For each ticket, the merchant name is usually the ticket **summary** itself (e.g., "Aposta Ganha", "Rhino", "CVC").
Use the summary as the merchant identifier unless it contains extra context like "Aposta Ganha - Starkbank - New Provider" (merchant = "Aposta Ganha", provider = "Starkbank").

Group tickets by merchant name.

### STEP 4 — Enrich from Slack (if available)

**Only attempt if** `SLACK_BOT_TOKEN` is set (check `source ~/.zshenv && [[ -n "$SLACK_BOT_TOKEN" ]]`).

If available, use MCP slack tools to search for each merchant name in TAM channels.
Pull thread counts and any notable conversations in the period.
If unavailable, skip silently — note "Slack: not configured" at bottom of report.

### STEP 5 — Enrich from Google Calendar (if available)

**Only attempt if** `GOOGLE_CREDENTIALS_PATH` is set (check `source ~/.zshenv && [[ -n "$GOOGLE_CREDENTIALS_PATH" ]]`).

If available, use MCP google-calendar tools to list events in the period.
Filter events where title or attendees match merchant names.
Categorize: QBR, kickoff call, technical review, support call.
If unavailable, skip silently — note "Calendar: not configured" at bottom of report.

### STEP 6 — Datadog metrics for LIVE merchants

For merchants with status "LIVE - EXTENDED CARE" or recently resolved tickets:

Use `mcp__plugin_yuno_datadog__query_metrics` to check if approval rate metrics exist.
Query window: last 7 days vs previous 7 days for any merchant with a recently resolved ticket.

If metric data is found, compute:
- `delta_approval` = recent_rate - previous_rate
- Confidence: HIGH if |delta| > 2%, MEDIUM if |delta| > 0.5%, LOW otherwise

If Datadog has no per-merchant tagging for approval rates, skip and note in report.

### STEP 7 — Generate report

Produce both layers. Always show the executive summary first, then the detailed breakdown.

---

## Report Format

### Executive Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TAM / Implementation Team — Impact Report
Period: [e.g., Feb 3 – Mar 5, 2026]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

PORTFOLIO SNAPSHOT
  [N] merchants active  |  [X] tickets  |  [Y]% proactive work

WORK DISTRIBUTION (TAM is not just bug fixes)
  Onboarding            [N]  [██████░░░░]  XX%
  Integration Support   [N]  [████░░░░░░]  XX%
  Health Check          [N]  [███░░░░░░░]  XX%
  Optimization          [N]  [██░░░░░░░░]  XX%
  Bug Fix / Incident    [N]  [█░░░░░░░░░]  XX%

MERCHANT HIGHLIGHTS
  [Merchant] — [WorkType] — [Status] — [TAM owner]
  ...

BUSINESS IMPACT (where data available)
  [Merchant] — approval rate [+X%] after [activity] — [confidence] confidence

MERCHANTS NEEDING ATTENTION
  [Merchant] — [reason: open >7d / declining metrics / no recent contact]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Detailed Breakdown

```
DETAILED TAM OPERATIONS — [Period]

BY OWNER
  [TAM Name]  |  [N] tickets  |  [X]% proactive  |  Merchants: [list]

ALL INTERVENTIONS
  [Key]    [Merchant]              [WorkType]           [Status]          [Assignee]
  IMP2-830 Aposta Ganha            INTEGRATION_SUPPORT  LIVE-EXTENDED     Pedro S.
  IMP2-610 Rhino                   ONBOARDING           Sandbox cert      Pedro S.
  ...

DATA SOURCES
  Jira: [N] tickets fetched from IMP2
  Datadog: [available / not available]
  Slack: [configured / not configured]
  Calendar: [configured / not configured]
```

---

## --publish flag

If the user included `--publish`, after generating the report in chat:

1. Check if a Confluence page titled "TAM Report — [Month Year]" already exists in the TAM or IMPLEMENTATION space using `searchConfluenceUsingCql`.
2. If it exists: update it with `updateConfluencePage`.
3. If not: create it with `createConfluencePage`.

Use the same content as the chat report but formatted as Confluence wiki markup.
Confirm to the user with the Confluence page URL when done.

---

## Tone and framing

- Always frame the report to **highlight proactive work** — onboarding, integration support, health checks are the majority of TAM work and that is the message.
- When showing work distribution, put proactive categories first.
- In the executive summary, open with the percentage of proactive work if it is >50%.
- Never say "only" or "just" when referring to bug fix count — state it neutrally.
- Business impact numbers should always include confidence level and a note that they are correlational, not causal.
