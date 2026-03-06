---
name: tam-login
description: |
  Configure authentication for all TAM report data sources.
  Triggers: "tam login", "set up TAM connections", "configure TAM",
  "tam auth", "tam setup", "configure tam credentials", "tam connections"
allowed-tools: Bash(source:*), Bash(sed:*), Bash(echo:*), Bash(grep:*),
  Bash(mkdir:*), Bash(test:*), Bash(cat:*), Read
---

# TAM Login Skill

Set up and verify all data source connections needed for the TAM executive report.
No terminal windows or shell scripts — everything happens in this chat.

## Constants

```
ZSHENV=~/.zshenv
GCAL_DEFAULT_PATH=~/.config/tam-metrics/gcal-client.json
```

## Execution Flow

### STEP 1 — Check current status

Run the following to check which connections are configured:

```bash
source ~/.zshenv

echo "=== TAM Connection Status ==="
echo ""

# Atlassian (always active via SSE OAuth)
echo "✓ Atlassian (Jira/Confluence) — MCP OAuth active, no setup needed"

# Datadog
if [[ -n "$DD_API_KEY" && -n "$DD_APP_KEY" ]]; then
  echo "✓ Datadog — configured"
else
  echo "✗ Datadog — not configured"
fi

# Slack
if [[ -n "$SLACK_CLIENT_ID" && -n "$SLACK_CLIENT_SECRET" ]]; then
  echo "✓ Slack — configured (browser auth fires on next Claude Code restart)"
elif [[ -n "$SLACK_BOT_TOKEN" ]]; then
  echo "✓ Slack — configured (bot token mode)"
else
  echo "✗ Slack — not configured"
fi

# Google Calendar
if [[ -n "$GOOGLE_CREDENTIALS_PATH" && -f "$GOOGLE_CREDENTIALS_PATH" ]]; then
  echo "✓ Google Calendar — configured (browser sign-in fires on first use)"
elif [[ -n "$GOOGLE_CREDENTIALS_PATH" ]]; then
  echo "~ Google Calendar — path set but file not found at: $GOOGLE_CREDENTIALS_PATH"
else
  echo "✗ Google Calendar — not configured"
fi
```

Show this status to the user clearly.

If **all connections are configured**: say so and stop. Suggest running `/tam-report`.

If **anything is missing**: proceed to configure each one below.

---

### STEP 2 — Datadog (if missing)

Datadog is managed by the existing yuno login system. Tell the user:

> "Datadog uses the existing yuno login. Run `/yuno-claude-plugin:login datadog` to configure it."

Do not configure Datadog inside this skill. Skip to Slack.

---

### STEP 3 — Slack (if missing)

Explain to the user:

> "For Slack, you need the Client ID and Client Secret from the shared Slack app.
> You can find these in 1Password under **'Claude TAM Report Slack App'**, or ask Pedro.
>
> If you don't have access to 1Password yet, ask Pedro to share:
> - The Slack app Client ID (looks like: `1234567890.987654321`)
> - The Slack app Client Secret (a long alphanumeric string)
>
> Paste the **Client ID** below:"

Wait for the user to provide the Client ID. Then ask for the Client Secret.

Once you have both values, save them:

```bash
source ~/.zshenv
sed -i '' '/^export SLACK_CLIENT_ID/d' ~/.zshenv 2>/dev/null
sed -i '' '/^export SLACK_CLIENT_SECRET/d' ~/.zshenv 2>/dev/null
echo "export SLACK_CLIENT_ID='REPLACE_WITH_VALUE'" >> ~/.zshenv
echo "export SLACK_CLIENT_SECRET='REPLACE_WITH_VALUE'" >> ~/.zshenv
source ~/.zshenv
echo "Slack credentials saved."
```

Replace `REPLACE_WITH_VALUE` with the actual values provided by the user.

Then confirm:
> "✓ Slack configured. A browser window will open the next time you restart Claude Code — just click Allow to complete the sign-in."

---

### STEP 4 — Google Calendar (if missing)

Explain to the user:

> "For Google Calendar, you need a credentials file called `gcal-client.json`.
> This is the shared app identity (not personal) — get it from 1Password under **'TAM Metrics GCal Client'**, or ask Pedro.
>
> Once you have the file, save it to:
> `~/.config/tam-metrics/gcal-client.json`
>
> Tell me the path where you saved it (press Enter to use the default above):"

Wait for the user's response. Use the default path if they press Enter or say nothing specific.

Expand `~` if present:
```bash
source ~/.zshenv
GCAL_PATH="${USER_PROVIDED_PATH:-$HOME/.config/tam-metrics/gcal-client.json}"
GCAL_PATH="${GCAL_PATH/#\~/$HOME}"

if [[ ! -f "$GCAL_PATH" ]]; then
  echo "File not found at: $GCAL_PATH"
  echo "Please copy gcal-client.json there first, then try again."
else
  mkdir -p "$(dirname "$GCAL_PATH")"
  sed -i '' '/^export GOOGLE_CREDENTIALS_PATH/d' ~/.zshenv 2>/dev/null
  echo "export GOOGLE_CREDENTIALS_PATH='${GCAL_PATH}'" >> ~/.zshenv
  source ~/.zshenv
  echo "Google Calendar configured."
fi
```

Then confirm:
> "✓ Google Calendar configured. A browser sign-in will open the first time you run `/tam-report` — just sign in with your Google account and click Allow. After that it's automatic."

---

### STEP 5 — Final status and next steps

Re-run the status check from STEP 1 and show the updated results.

Then tell the user:

> **You're all set!**
>
> - If Slack was just configured: restart Claude Code for browser auth to trigger.
> - Then run `/tam-report` to generate your first executive report.
>
> To generate a report now (without Slack/Calendar if just configured):
> `/tam-report`

---

## Fallback: Slack bot token mode

If the user cannot access the Slack app credentials (no 1Password access, Pedro unavailable),
offer the fallback:

> "As a simpler alternative, you can use a shared Slack bot token.
> Ask Pedro for the `SLACK_BOT_TOKEN` from 1Password. Paste it below:"

Save via:
```bash
sed -i '' '/^export SLACK_BOT_TOKEN/d' ~/.zshenv 2>/dev/null
echo "export SLACK_BOT_TOKEN='REPLACE_WITH_VALUE'" >> ~/.zshenv
source ~/.zshenv
echo "Slack bot token saved."
```

Note: bot token mode reads all channels the bot is in — simpler but less personal than OAuth.
