---
name: slack-notify
description: Send Slack notifications via Bot Token (chat.postMessage). Use when you want to notify the user about completed tasks, errors, important results, or any status updates worth surfacing in Slack. Supports sending to any channel or user DM. Triggers on phrases like "send slack", "notify slack", "slack message", "ping me on slack", "send notification".
metadata:
  internal: true
---

# Slack Notify Skill

Sends a message to Slack using a Bot Token and the `chat.postMessage` API.
Supports any recipient: channels, group DMs, or individual users.

## Configuration

Config is loaded in this order — project-level overrides global:

1. `~/.config/opencode/.env` — global defaults (token, personal IDs)
2. `.env.local` in the repo root — project-specific overrides

**Global config** (`~/.config/opencode/.env`):
```
SLACK_BOT_TOKEN=xoxb-...       # Bot OAuth Token
SLACK_USER_ME=...              # Your own Slack user ID
SLACK_USER_<NAME>=...          # Other people, e.g. SLACK_USER_JENS=...
```

**Project config** (`.env.local`, optional overrides):
```
SLACK_DEFAULT_CHANNEL=...      # Project-specific default channel ID
SLACK_USER_<NAME>=...          # Project-specific recipients
```

If `SLACK_BOT_TOKEN` is missing, tell the user and show the setup instructions below.
Never hardcode user IDs or channel IDs — always use env vars.

## Recipient resolution

When the user says "send to Jens" or "notify Jens", resolve the recipient by looking up
`SLACK_USER_JENS`. The mapping is: uppercase the name → prepend `SLACK_USER_`.

```
"send to Jens"  → $SLACK_USER_JENS
"ping me"       → $SLACK_USER_ME
"notify Alice"  → $SLACK_USER_ALICE
no recipient    → $SLACK_DEFAULT_CHANNEL
```

If the variable is not set, tell the user to add `SLACK_USER_<NAME>=<slack-user-id>`
to `~/.config/opencode/.env` (global) or `.env.local` (project-specific).

## Workflow

### 1. Compose the message

Keep messages concise and actionable. Always mention that you are Josie (the AI coding
agent) so recipients know who sent it. Use Slack markdown where helpful:
- `*bold*` for emphasis
- `` `code` `` for file paths, commands, identifiers
- `>` for blockquotes / context

### 2. Send via bash

```bash
# Load global config, then project config (project overrides global)
set -a
[[ -f ~/.config/opencode/.env ]] && source ~/.config/opencode/.env
[[ -f .env.local ]] && source .env.local
set +a

# Send to default channel
PAYLOAD=$(jq -n --arg text "YOUR MESSAGE" --arg channel "$SLACK_DEFAULT_CHANNEL" \
  '{channel: $channel, text: $text}')
RESPONSE=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")
echo "$RESPONSE" | jq -r 'if .ok then "Sent!" else "ERROR: \(.error)" end'
```

To send to a specific recipient, replace `$SLACK_DEFAULT_CHANNEL` with the appropriate var:

```bash
--arg channel "$SLACK_USER_ME"     # ping yourself
--arg channel "$SLACK_USER_JENS"   # ping Jens
```

### 3. Confirm delivery

The API always returns HTTP 200. Check the JSON `ok` field — `true` means delivered,
`false` means check the `error` field (e.g. `channel_not_found`, `not_in_channel`).

## When to use proactively

- After a long-running task completes (build, migration, test run)
- When a critical error or anomaly is detected during automated work
- When explicitly asked by the user to notify them

## Setup: Create a Slack Bot Token

1. Go to **https://api.slack.com/apps** → **Create New App** → **From scratch**
2. Name the app (e.g. `elhico-notify`), pick your workspace → **Create App**
3. Left sidebar: **OAuth & Permissions**
4. Scroll to **Scopes → Bot Token Scopes** → add **`chat:write`**
   - Also add **`chat:write.public`** to post to channels without an invite
5. Scroll up → **Install to Workspace** → **Allow**
6. Copy the **Bot User OAuth Token** (`xoxb-...`) → add to `~/.config/opencode/.env`
7. **Invite the bot to channels** (not needed for DMs):
   In Slack: open the channel → `/invite @your-bot-name`

> **Finding a Slack user ID**: Click the person's name → View profile → three-dot menu
> → Copy member ID. Format: `U012AB3CD`.
