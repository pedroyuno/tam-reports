# TAM Metrics

Executive reporting system for the Yuno TAM (Technical Account Manager) team — built to change the narrative that TAM only fixes bugs.

## What it does

- **Categorizes TAM work** into 8 types (onboarding, optimization, health checks, education, and more — not just bug fixes)
- **Correlates interventions with business impact** — approval rate deltas and estimated transactions recovered per merchant
- **Two-layer reports**: executive summary for C-level/VPs + detailed breakdown for the team
- **On-demand via CLI** and **weekly automated** delivery to Confluence

## Data Sources

| Source | MCP | Contributes |
|--------|-----|-------------|
| Jira | `mcp__plugin_yuno_atlassian` | Tickets, worklog, merchant context |
| Datadog | `mcp__plugin_yuno_datadog` | Approval rates and error metrics before/after intervention |
| Slack | `korotovsky/slack-mcp-server` | Channel threads, merchant conversations |
| Google Calendar | `nspady/google-calendar-mcp` | QBRs, onboarding calls, time investment per merchant |

## Architecture

A single Claude Code skill (`yuno:tam-report`) orchestrates all four MCPs. No Python agent or backend infrastructure needed.

```
/tam-report
  → Fetch Jira tickets (TAM project, last 30d)
  → Classify each ticket into WorkType
  → Extract merchant context
  → Enrich from Slack threads
  → Enrich from Google Calendar events
  → Correlate with Datadog metrics (before/after windows)
  → Generate executive summary + detailed breakdown
```

## Work Type Taxonomy

| Type | Category |
|------|----------|
| `INCIDENT_RESPONSE` | Reactive |
| `BUG_FIX_ESCALATION` | Reactive |
| `INTEGRATION_SUPPORT` | Proactive |
| `OPTIMIZATION` | Proactive |
| `HEALTH_CHECK` | Proactive |
| `ONBOARDING` | Proactive |
| `CONFIGURATION` | Proactive |
| `EDUCATION` | Proactive |

Target: reactive work should be <30% of total, proving TAM drives proactive merchant value.

## Business Impact Correlation

For each resolved ticket, the skill computes a before/after metric delta:

```
baseline window: [resolution_date - 30d, resolution_date - 7d]
post window:     [resolution_date + 7d,  resolution_date + 37d]

HIGH confidence   → delta > 2%  and post_days >= 14
MEDIUM confidence → delta > 0.5% and post_days >= 7
```

The executive report only surfaces HIGH/MEDIUM confidence attributions.

## Setup

### 1. Add Slack MCP

```bash
# Add to ~/.zshenv
export SLACK_BOT_TOKEN=xoxb-...   # scopes: channels:history, search:messages, users:read
```

Add to `.mcp.json`:
```json
"slack": {
  "command": "npx",
  "args": ["-y", "@korotovsky/slack-mcp-server"],
  "env": { "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}" }
}
```

### 2. Add Google Calendar MCP

```bash
# One-time OAuth flow — saves credentials to:
export GOOGLE_CREDENTIALS_PATH=/Users/pedro/.config/tam-metrics/gcal-credentials.json
```

Add to `.mcp.json`:
```json
"google-calendar": {
  "command": "npx",
  "args": ["-y", "@nspady/google-calendar-mcp"],
  "env": { "GOOGLE_CREDENTIALS_PATH": "${GOOGLE_CREDENTIALS_PATH}" }
}
```

### 3. Install the skill

Copy `skills/tam-report/SKILL.md` into the yuno plugin:
```bash
cp -r skills/tam-report ~/.claude/plugins/cache/yuno-plugins/yuno/0.0.17/skills/
```

## Usage

```bash
/tam-report                        # Last 30 days, both report layers
/tam-report --period 90d           # Last quarter
/tam-report --tam pedro@yuno.com   # Filter by TAM owner
/tam-report --merchant acme        # Focus on one merchant
/tam-report --publish              # Also push to Confluence
```

### Weekly automation

```bash
# Add to crontab (Monday 9am)
0 9 * * 1 /bin/zsh -c 'source ~/.zshenv && claude -p "/tam-report --publish"'
```

## Report Format

### Executive Summary

```
TAM Impact — [Period]
Portfolio: N merchants · X interventions · Y% proactive work

Business Impact (High Confidence)
Merchant      | Activity             | Approval Rate | Est. Txns
Acme Corp     | Routing optimization | +3.2%         | ~14,000
Beta Inc      | Webhook retry fix    | +1.8%         | ~6,400

Work Distribution
Onboarding 25% · Optimization 22% · Integration Support 19% · ...
```

### Detailed Breakdown

Per-TAM table, all interventions with confidence levels, Slack activity summary, calendar summary, and full metric attribution methodology.

## Files

```
PLAN.md          — Full implementation plan and build order
skills/          — Claude Code skill (to be built)
```

## References

- [Slack MCP Server](https://github.com/korotovsky/slack-mcp-server)
- [Google Calendar MCP](https://github.com/nspady/google-calendar-mcp)
- [Google Official MCP for Google Services](https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services)
