---
name: slack-notify
description: Send Slack notifications via Bot Token (chat.postMessage). Use when you want to notify the user about completed tasks, errors, important results, or any status updates worth surfacing in Slack. Supports sending to any channel or user DM. Triggers on phrases like "send slack", "notify slack", "slack message", "ping me on slack", "send notification".
---

# Slack Notify Skill

Sends a message to Slack using a Bot Token and the `chat.postMessage` API.
Supports any recipient: channels, group DMs, or individual users.

## Prerequisites

These variables must be set in `.env.local` at the repo root:

```
SLACK_BOT_TOKEN=xoxb-...      # Bot OAuth Token
SLACK_DEFAULT_CHANNEL=...     # Default recipient (channel ID or user ID)
SLACK_USER_ME=...             # The current user's own Slack ID (for "ping me" requests)
```

Additional users follow the pattern `SLACK_USER_<NAME>=...`, e.g.:
```
SLACK_USER_BOB=...
SLACK_USER_ALICE=...
```

If `SLACK_BOT_TOKEN` is missing, tell the user and show the setup instructions below.
Look up recipient IDs from the env vars — never hardcode them in messages or scripts.

## Recipient resolution

When the user says "send to Bob" or "notify Bob", resolve the recipient by looking up
`SLACK_USER_BOB` from `.env.local`. The mapping is: uppercase the name → prepend `SLACK_USER_`.

```bash
set -a && source .env.local && set +a

# "send to Bob"  → $SLACK_USER_BOB
# "ping me"       → $SLACK_USER_ME
# "notify Alice"  → $SLACK_USER_ALICE
# no recipient    → $SLACK_DEFAULT_CHANNEL
```

If the variable is not set, tell the user to add `SLACK_USER_<NAME>=<slack-user-id>` to `.env.local`.

## Workflow

### 1. Compose the message

Keep messages concise and actionable. Always mention that you are Josie (the AI coding agent) so recipients know who sent it. Use Slack markdown where helpful:
- `*bold*` for emphasis
- `` `code` `` for file paths, commands, identifiers
- `>` for blockquotes / context

### 2. Send via bash

Run this inline — no external script needed:

```bash
# Load .env.local
set -a && source .env.local && set +a

# Send to default channel
PAYLOAD=$(jq -n --arg text "YOUR MESSAGE" --arg channel "$SLACK_DEFAULT_CHANNEL" '{channel: $channel, text: $text}')
RESPONSE=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")
echo "$RESPONSE" | jq -r 'if .ok then "Sent!" else "ERROR: \(.error)" end'
```

To send to a specific recipient, replace `$SLACK_DEFAULT_CHANNEL` with the appropriate env var:

```bash
--arg channel "$SLACK_USER_ME"    # ping yourself
--arg channel "$SLACK_USER_BOB"  # ping Bob
```

Or use the existing script if present:

```bash
./tools/slack-notify.sh "message"
./tools/slack-notify.sh --to "$SLACK_USER_BOB" "message"
./tools/slack-notify.sh --to "$SLACK_USER_ME" "message"
```

### 3. Confirm delivery

The API always returns HTTP 200. Check the JSON `ok` field — `true` means delivered, `false` means check the `error` field (e.g. `channel_not_found`, `not_in_channel`).

## When to use proactively

- When explicitly asked by the user to notify them

## Setup: Create a Slack Bot Token

1. Go to **https://api.slack.com/apps** → **Create New App** → **From scratch**
2. Name the app (e.g. `elhico-notify`), pick your workspace → **Create App**
3. Left sidebar: **OAuth & Permissions**
4. Scroll to **Scopes → Bot Token Scopes** → add **`chat:write`**
   - Also add **`chat:write.public`** to post to channels without an invite
5. Scroll up → **Install to Workspace** → **Allow**
6. Copy the **Bot User OAuth Token** (`xoxb-...`) → add to `.env.local`
7. **Invite the bot to channels** (not needed for DMs):
   In Slack: open the channel → `/invite @your-bot-name`

> **Finding a Slack user ID**: Click the person's name → View profile → three-dot menu → Copy member ID. Format: `U012AB3CD`.


## Workflow

### 1. Compose the message

Keep messages concise and actionable. Always mention that you are Josie (the AI coding agent) so recipients know who sent it. Use Slack markdown where helpful:
- `*bold*` for emphasis
- `` `code` `` for file paths, commands, identifiers
- `>` for blockquotes / context

### 2. Send via bash

Run this inline — no external script needed:

```bash
# Load .env.local
set -a && source .env.local && set +a

# Send to default channel
PAYLOAD=$(jq -n --arg text "YOUR MESSAGE" --arg channel "$SLACK_DEFAULT_CHANNEL" '{channel: $channel, text: $text}')
RESPONSE=$(curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD")
echo "$RESPONSE" | jq -r 'if .ok then "Sent!" else "ERROR: \(.error)" end'
```

To send to a specific recipient, replace `$SLACK_DEFAULT_CHANNEL` with the target:

```bash
# To Bob
--arg channel "$SLACK_USER_BOB"

# To David
--arg channel "$SLACK_USER_DAVID"

# To any channel or user ID directly
--arg channel "C0AJQ21U01L"
--arg channel "U7EJWLVN0"
```

Or use the existing script if present:

```bash
./tools/slack-notify.sh "message"
./tools/slack-notify.sh --to "U7EJWLVN0" "message"
```

### 3. Confirm delivery

The API always returns HTTP 200. Check the JSON `ok` field — `true` means delivered, `false` means check the `error` field (e.g. `channel_not_found`, `not_in_channel`).

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
6. Copy the **Bot User OAuth Token** (`xoxb-...`) → add to `.env.local`
7. **Invite the bot to channels** (not needed for DMs):
   In Slack: open the channel → `/invite @your-bot-name`

> **Finding a Slack user ID**: Click the person's name → View profile → three-dot menu → Copy member ID. Format: `U012AB3CD`.
