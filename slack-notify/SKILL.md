---
name: slack-notify
description: Send Slack notifications and files via Bot Token (chat.postMessage, files.uploadV2). Use when you want to notify the user about completed tasks, errors, important results, or any status updates worth surfacing in Slack. Supports sending to any channel or user DM. Can also upload files (logs, reports, code snippets, etc.). Triggers on phrases like "send slack", "notify slack", "slack message", "ping me on slack", "send notification", "send file to slack", "upload to slack", "slack file".
metadata:
  internal: true
---

# Slack Notify Skill

Sends messages and files to Slack using a Bot Token.
- **Messages** via `chat.postMessage` API
- **Files** via `files.getUploadURLExternal` + `files.completeUploadExternal` API (Slack's current upload flow)

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

**IMPORTANT — Multiline messages:** `jq --arg` does NOT handle literal newlines
in shell strings correctly (causes JSON parse errors). For messages with line breaks,
use `python3` with `json.dumps()` which properly escapes newlines for JSON.

**For short single-line messages** — use jq:

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

**For multiline messages** (preferred) — use python3:

```bash
set -a
[[ -f ~/.config/opencode/.env ]] && source ~/.config/opencode/.env
[[ -f .env.local ]] && source .env.local
set +a

python3 -c "
import json, os, urllib.request

message = '''YOUR MULTILINE
MESSAGE HERE'''

payload = json.dumps({
    'channel': os.environ['SLACK_DEFAULT_CHANNEL'],
    'text': message
}).encode('utf-8')

req = urllib.request.Request(
    'https://slack.com/api/chat.postMessage',
    data=payload,
    headers={
        'Authorization': f\"Bearer {os.environ['SLACK_BOT_TOKEN']}\",
        'Content-Type': 'application/json'
    }
)
resp = urllib.request.urlopen(req)
result = json.loads(resp.read())
print('Sent!' if result.get('ok') else f\"ERROR: {result.get('error')}\")
"
```

To send to a specific recipient, replace `SLACK_DEFAULT_CHANNEL` with the appropriate var:

```python
os.environ['SLACK_USER_ME']        # ping yourself
os.environ['SLACK_USER_JENS']      # ping Jens
```

### 3. Confirm delivery

The API always returns HTTP 200. Check the JSON `ok` field — `true` means delivered,
`false` means check the `error` field (e.g. `channel_not_found`, `not_in_channel`).

### 4. Upload a file

Slack uses a two-step upload flow. First you request an upload URL, then you upload the
file content there, and finally you complete the upload by associating it with a channel.

**Step 1 — Get an upload URL:**

```bash
# Load config
set -a
[[ -f ~/.config/opencode/.env ]] && source ~/.config/opencode/.env
[[ -f .env.local ]] && source .env.local
set +a

FILE_PATH="/path/to/your/file.txt"
FILE_NAME=$(basename "$FILE_PATH")
FILE_SIZE=$(wc -c < "$FILE_PATH" | tr -d ' ')

URL_RESPONSE=$(curl -s -X GET "https://slack.com/api/files.getUploadURLExternal" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -G \
  --data-urlencode "filename=${FILE_NAME}" \
  --data-urlencode "length=${FILE_SIZE}")

UPLOAD_URL=$(echo "$URL_RESPONSE" | jq -r '.upload_url')
FILE_ID=$(echo "$URL_RESPONSE" | jq -r '.file_id')

if [ "$UPLOAD_URL" = "null" ]; then
  echo "ERROR: $(echo "$URL_RESPONSE" | jq -r '.error')"
  exit 1
fi
```

**Step 2 — Upload the file content:**

```bash
curl -s -X POST "$UPLOAD_URL" \
  -F "file=@${FILE_PATH}"
```

**Step 3 — Complete the upload and share to a channel:**

```bash
CHANNEL_ID="$SLACK_DEFAULT_CHANNEL"  # must be a channel ID (C...), NOT a user ID (U...)
INITIAL_COMMENT="Here's the file you requested"  # optional message

COMPLETE_PAYLOAD=$(jq -n \
  --arg file_id "$FILE_ID" \
  --arg channel "$CHANNEL_ID" \
  --arg comment "$INITIAL_COMMENT" \
  '{files: [{id: $file_id, title: "My File"}], channel_id: $channel, initial_comment: $comment}')

COMPLETE_RESPONSE=$(curl -s -X POST "https://slack.com/api/files.completeUploadExternal" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "$COMPLETE_PAYLOAD")

echo "$COMPLETE_RESPONSE" | jq -r 'if .ok then "File uploaded!" else "ERROR: \(.error)" end'
```

**IMPORTANT — channel_id must be a Channel ID (format `C...`), not a User ID (`U...`).**
The `files.completeUploadExternal` API requires a channel/conversation ID matching the
regex `^[CGDZ][A-Z0-9]{8,}$`. To upload to a user DM, you would need to first open a
DM conversation via `conversations.open` (requires `im:write` scope). If that scope is
not available, use `$SLACK_DEFAULT_CHANNEL` instead and mention/tag the user in the
`initial_comment`.

**Uploading text content directly (without an existing file):**

If you need to send generated content (e.g. a log snippet, report, or code) as a file
without writing it to disk first, write it to a temp file and upload that:

```bash
CONTENT="your generated content here"
TMPFILE=$(mktemp /tmp/slack-upload-XXXXXX.txt)
echo "$CONTENT" > "$TMPFILE"

# Then use the same 3-step flow above with FILE_PATH="$TMPFILE"
# Clean up after:
rm -f "$TMPFILE"
```

**Notes on file uploads:**
- The `initial_comment` in Step 3 is optional — omit it if no message is needed
- Always mention you are Josie (the AI coding agent) in the `initial_comment`
- Supported: any file type Slack accepts (text, images, PDFs, archives, etc.)
- Max file size depends on the workspace plan (free: 1 GB total storage)

## When to use proactively

- After a long-running task completes (build, migration, test run)
- When a critical error or anomaly is detected during automated work
- When explicitly asked by the user to notify them
- When the user asks to share a file, log output, report, or code snippet via Slack

## Setup: Create a Slack Bot Token

1. Go to **https://api.slack.com/apps** → **Create New App** → **From scratch**
2. Name the app (e.g. `elhico-notify`), pick your workspace → **Create App**
3. Left sidebar: **OAuth & Permissions**
4. Scroll to **Scopes → Bot Token Scopes** → add **`chat:write`**
   - Also add **`chat:write.public`** to post to channels without an invite
   - Also add **`files:write`** to upload files to channels and DMs
   - Also add **`files:read`** if you want the bot to read/access uploaded files later
5. Scroll up → **Install to Workspace** → **Allow**
6. Copy the **Bot User OAuth Token** (`xoxb-...`) → add to `~/.config/opencode/.env`
7. **Invite the bot to channels** (not needed for DMs):
   In Slack: open the channel → `/invite @your-bot-name`

> **Finding a Slack user ID**: Click the person's name → View profile → three-dot menu
> → Copy member ID. Format: `U012AB3CD`.
