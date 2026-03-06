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

### STEP 1b — Fetch full ticket details with comments

For each ticket key returned in STEP 1, call `getJiraIssue` to retrieve the full description and all comments.

Limit: fetch details for up to 20 tickets, prioritizing the most recently updated.
If there are more than 20 tickets, fetch details only for those updated within the period.

From the comments and description, extract:
- What activities were performed (configuration, testing, meetings, escalations, follow-ups)
- Which parties were involved (merchant, internal teams, third-party providers)
- What problems were encountered and how they were resolved
- Technical specifics: provider names, payment methods, error types, credentials, webhooks

**This is the storytelling gold mine.** TAM members document actual work in comments — "joined merchant call", "opened ticket with NuPay", "sent webhook IPs to provider", "resolved credential issue". These become the narrative bullets.

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

### STEP 3b — Write merchant narrative

For each merchant group, synthesize a 3–5 bullet narrative using:
- Full descriptions and comments from STEP 1b
- WorkType(s) from STEP 2
- Ticket status and resolution state

**Bullet format** — always start with an active verb:
```
"Supported [merchant] in configuring [provider] and resolving [issue]"
"Coordinated with [team/provider] to troubleshoot [problem]"
"Led configuration and testing of [integration], including [detail]"
"Opened and followed up on [N] Jira tickets for [issue type]"
"Participated in [N] meetings and provided guidance on [topic]"
"Assisted with [specific task, e.g., org ID retrieval, credential validation]"
```

Consolidate related comments into single bullets — don't list every comment individually.
If no comments are available, derive activities from ticket description and status transitions.

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

### STEP 6 — Estimate business impact per merchant

For each merchant, determine impact using this priority order:

**A) Datadog data (actual numbers):**
Use `mcp__plugin_yuno_datadog__query_metrics` for merchants with resolved tickets or LIVE status.
Query approval rate metrics with merchant name as tag filter.
Compare: period window vs 30 days prior.
If delta found:
- `delta_approval` = recent_rate - previous_rate
- Confidence: HIGH if |delta| > 2%, MEDIUM if |delta| > 0.5%, LOW otherwise
- Report as: `approval rate +X% (confirmed, HIGH confidence)`

**B) Stage-based estimate (when Datadog has no per-merchant data):**
Use work type + ticket status to produce a qualitative estimate with range:

| Merchant stage           | WorkType             | Estimated impact                                              |
|--------------------------|----------------------|---------------------------------------------------------------|
| New go-live (Done/Live)  | ONBOARDING           | "Unlocked ~$X TPV/month" (estimate based on typical new merchant range: $50k–$500k/mo) |
| New payment method live  | INTEGRATION_SUPPORT  | "Added new payment method coverage, est. +$X TPV/month"      |
| Extended care, stable    | HEALTH_CHECK         | "Maintained $X TPV at healthy baseline"                       |
| Active issue resolved    | BUG_FIX_ESCALATION   | "Restored service, preventing ~N hours of degraded checkout"  |
| Optimization completed   | OPTIMIZATION         | "Approval rate improvement est. +1–3% (estimate, unconfirmed)"|

Always label estimates with `(est.)`. Use $50k–$150k/month for typical small merchants, $200k–$500k for mid-size, $500k+ for large. When uncertain, use the conservative end or omit the number.

**C) Qualitative only (fallback):**
When no estimate is appropriate, use a plain impact statement:
- "Integration unblocked → go-live enabled"
- "Project continuity ensured, no delay"
- "Technical blocker resolved, merchant unblocked"

An impact line is always present for every merchant — never leave it blank.

**Aggregate:** After processing all merchants, sum estimated TPV influenced and count integrations unblocked/go-lives enabled. Use these in the executive summary.

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
  Est. business impact: ~$[Z]M TPV influenced  |  [N] integrations unblocked

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
  [Merchant] — ~$X TPV/month unlocked (est.) — go-live enabled

MERCHANTS NEEDING ATTENTION
  [Merchant] — [reason: open >7d / declining metrics / no recent contact]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Merchant Stories

Placed immediately after the executive summary. One entry per merchant, ordered by business impact (highest first).

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MERCHANT STORIES — What TAM actually did
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[Merchant Name]  ([Provider/Integration] · [WorkType] · [TAM owner])
  • [Active verb] [what was done, specific and factual]
  • [Active verb] [coordination, troubleshooting, or follow-up activity]
  • [Active verb] [technical detail or outcome]
  Impact: [impact statement with number if available, or qualitative]

Example:
Águia Branca  (NuPay/PIX · INTEGRATION_SUPPORT · Pedro S.)
  • Supported configuration of PIX for Santander and resolved NuPay credential issues,
    including opening and following up on Jira tickets with internal teams.
  • Coordinated with internal teams and the merchant to troubleshoot integration blockers.
  • Assisted with org ID retrieval and clarified technical flows for the merchant.
  Impact: Integration unblocked → go-live enabled. Est. ~$120k TPV/month unlocked. (est.)

Aposta Ganha  (Pagsmile & Okto · ONBOARDING · Pedro S.)
  • Led configuration and testing of Pagsmile and Okto integrations.
  • Provided production webhook and IP information to both providers.
  • Coordinated merchant and internal teams for go-live readiness.
  Impact: 2 new payment methods enabled. Est. +$80k TPV/month. (est.)

[... all merchants ...]
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
  Note: estimates marked (est.) are not from data — correlational, not causal.
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

### Storytelling rules (MERCHANT STORIES section)

- Bullet points must start with active verbs: "Led", "Supported", "Coordinated", "Resolved", "Opened", "Provided", "Participated in", "Assisted with", "Ensured"
- Keep bullets factual and grounded in what Jira comments actually say — do not invent activities
- Consolidate related comments into one bullet — don't repeat the same idea twice
- The Impact line is mandatory for every merchant, even if qualitative
- Estimates are always labeled `(est.)` — never present a number without this label unless it comes from Datadog
- Order merchants by impact magnitude (highest first) within the stories section
