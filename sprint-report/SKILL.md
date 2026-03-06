---
name: sprint-report
description: Generate a sprint/iteration report for a GitHub repository. Analyzes commits, issues, PRs, LOC changes, contributors, hotspots, refactoring candidates, knowledge silos, temporal coupling, bug hotspots, and codebase growth. Triggers on phrases like "sprint report", "sprint auswertung", "sprint summary", "sprint stats", "analyze sprint", "iteration report".
metadata:
  internal: true
---

# Sprint Report Skill

Generates a comprehensive sprint/iteration report for a GitHub repository by analyzing
git history and GitHub data (issues, PRs) within a user-specified time range.

## Prerequisites

- `git` CLI available and repo cloned locally (for git-based metrics)
- `gh` CLI authenticated (for GitHub API metrics: issues, PRs)
- `jq` available for JSON processing

## Input Parameters

When triggered, **always ask the user** for:

1. **Time range** — ask interactively. Accept flexible formats:
   - Explicit dates: `2026-02-17` to `2026-03-06`
   - Relative: `last 2 weeks`, `last month`, `last 30 days`
   - Sprint name (just used as label): `Sprint 42`
   - Convert relative ranges to ISO dates for git/gh commands

2. **Repository** (optional) — default: current working directory's git remote.
   User can override with `owner/repo` format. If overridden, use `gh` commands
   with `--repo owner/repo` flag. For git log commands, the repo must be cloned locally.

3. **Output format** — ask if not specified. **Multiple selections allowed** (e.g. `slack + file`):
   - `terminal` (default) — render report as markdown in chat
   - `slack` — send via slack-notify skill (condensed summary + full .md upload + chart PNGs)
   - `file` — save as `sprint-report-YYYY-MM-DD.md` in repo root
   
   The user can pick any combination. If multiple formats are selected, execute all of them
   in sequence. For example: `slack + file` generates the report once, then sends it to Slack
   AND saves the .md file locally. `terminal + slack` renders in chat AND sends to Slack.

## Date Variable Convention

Throughout this skill, use these bash variables:
```bash
SINCE="2026-02-17"   # Sprint start (inclusive)
UNTIL="2026-03-06"   # Sprint end (inclusive)
REPO_OWNER_NAME=""   # Empty = current repo, or "owner/repo"
SPRINT_LABEL="Sprint 42"  # Display label
```

## Metrics & Data Collection

Collect all metrics using the bash commands below. Run them sequentially, storing
results in variables. Then assemble the final report.

**Important**: For all `gh` commands, append `--repo "$REPO_OWNER_NAME"` only if
`REPO_OWNER_NAME` is non-empty. For git commands, always operate on the local clone.

---

### 1. Commits Overview

Total commits, grouped by author.

```bash
# Total commits in range
git log --oneline --after="$SINCE" --before="$UNTIL" | wc -l

# Commits per author
git log --after="$SINCE" --before="$UNTIL" --format='%aN' | sort | uniq -c | sort -rn
```

---

### 2. Lines of Code (Added / Removed)

```bash
# Total LOC added/removed
git log --after="$SINCE" --before="$UNTIL" --numstat --format='' | \
  awk '{ added += $1; removed += $2 } END { print "+"added, "-"removed }'

# LOC per author
git log --after="$SINCE" --before="$UNTIL" --numstat --format='AUTHOR:%aN' | \
  awk '/^AUTHOR:/ { author=$0; sub(/^AUTHOR:/, "", author); next }
       NF==3 { added[author]+=$1; removed[author]+=$2 }
       END { for (a in added) print a": +"added[a], "-"removed[a] }' | sort -t'+' -k2 -rn
```

---

### 3. Issues

```bash
# Issues created in range
gh issue list --state all --search "created:${SINCE}..${UNTIL}" --limit 500 \
  --json number,title,state,labels,createdAt,closedAt

# Issues closed in range
gh issue list --state closed --search "closed:${SINCE}..${UNTIL}" --limit 500 \
  --json number,title,labels,createdAt,closedAt

# Issues still open (created in range, not yet closed)
gh issue list --state open --search "created:${SINCE}..${UNTIL}" --limit 500 \
  --json number,title,labels,createdAt
```

---

### 4. Issue Labels Distribution

From the issues collected above, aggregate by label:

```bash
# Count issues per label (from the created-in-range set)
gh issue list --state all --search "created:${SINCE}..${UNTIL}" --limit 500 --json labels \
  | jq -r '.[].labels[].name' | sort | uniq -c | sort -rn
```

---

### 5. Issue Cycle Time

Average time from issue creation to close, for issues closed in the sprint:

```bash
gh issue list --state closed --search "closed:${SINCE}..${UNTIL}" --limit 500 \
  --json number,title,createdAt,closedAt | jq -r '
  [.[] | {
    number,
    title,
    days: ((((.closedAt | fromdateiso8601) - (.createdAt | fromdateiso8601)) / 86400) | floor)
  }] | sort_by(-.days) |
  (map(.days) | add / length | floor) as $avg |
  "Average cycle time: \($avg) days\n" +
  (.[0:10][] | "#\(.number) \(.title): \(.days) days")
'
```

---

### 6. Top Contributors

Combine commits + LOC into a ranking table. Use data from metrics 1 and 2.
Present as a markdown table sorted by commits descending.

---

### 7. Hotspots (Most Changed Files)

Files with the most commits (change frequency) in the sprint:

```bash
# Top 15 most frequently changed files
git log --after="$SINCE" --before="$UNTIL" --name-only --format='' | \
  sort | uniq -c | sort -rn | head -20
```

---

### 8. Refactoring Candidates (Churn Analysis)

Files with high churn = lots of lines added AND removed (code being rewritten).
High churn on frequently changed files = refactoring candidate.

```bash
# Churn per file: files with high add+delete ratio
git log --after="$SINCE" --before="$UNTIL" --numstat --format='' | \
  awk '$1 != "-" { files[$3] += $1 + $2; added[$3] += $1; removed[$3] += $2 }
       END { for (f in files) if (files[f] > 20) print files[f], "churn", "(+"added[f], "-"removed[f]")", f }' | \
  sort -rn | head -15
```

Interpretation guide for the report:
- High churn (many lines added AND removed) = code instability, potential refactoring candidate
- High churn on a hotspot file = **priority refactoring target**
- Flag files that appear in BOTH hotspots AND churn top-15

---

### 9. Knowledge Distribution / Bus Factor

For the top hotspot files, check how many unique authors have touched them.
Files with only 1 author = knowledge silo = risk.

```bash
# For top 10 hotspot files, count unique authors
# IMPORTANT: --pretty=format:'%aN' can leak commit body/metadata in some repos.
# Use git rev-list + xargs for clean author extraction.
HOTSPOT_FILES=$(git log --after="$SINCE" --before="$UNTIL" --name-only --pretty=format:'' | \
  grep -v '^$' | sort | uniq -c | sort -rn | head -10)

echo "$HOTSPOT_FILES" | while read -r COUNT FILE; do
  [ -z "$FILE" ] && continue
  # Clean author extraction: rev-list for hashes, then per-hash author lookup
  NAMES=$(git rev-list --after="$SINCE" --before="$UNTIL" HEAD -- "$FILE" | \
    xargs -I{} git log -1 --format='%aN' {} | sort -u | paste -sd', ' -)
  AUTHORS_COUNT=$(git rev-list --after="$SINCE" --before="$UNTIL" HEAD -- "$FILE" | \
    xargs -I{} git log -1 --format='%aN' {} | sort -u | grep -c .)
  RISK="OK"
  [ "$AUTHORS_COUNT" -le 1 ] && RISK="SILO"
  echo "$COUNT changes | $AUTHORS_COUNT author(s) | $FILE | $NAMES | $RISK"
done
```

Interpretation:
- 1 author on a hotspot = **knowledge silo / bus factor risk** (highlight in report)
- 2+ authors = shared knowledge (healthy)

---

### 10. Temporal Coupling

Files that are frequently changed together in the same commit = hidden dependencies.

```bash
# Find files commonly changed together
# NOTE: use --pretty=format: not --format= to avoid issues with some git versions
git log --after="$SINCE" --before="$UNTIL" --name-only --pretty=format:'COMMIT_SEP' | \
  awk '
  /^COMMIT_SEP$/ {
    for (i=0; i<n; i++) for (j=i+1; j<n; j++) {
      pair = (files[i] < files[j]) ? files[i] " <-> " files[j] : files[j] " <-> " files[i]
      count[pair]++
    }
    n=0; next
  }
  NF > 0 && !/^$/ { files[n++] = $0 }
  END {
    for (p in count) if (count[p] >= 3) print count[p], p
  }' | sort -rn | head -15
```

Interpretation:
- Files always changed together MAY indicate:
  - Missing abstraction (should be one module)
  - Tight coupling (consider refactoring to reduce dependency)
  - Normal co-change (e.g., test + implementation) — not always bad
- Flag pairs that are NOT obvious test+impl pairs

---

### 11. Bug Hotspots

Files that appear in commits linked to bug-labeled issues.

```bash
# Get bug issue numbers
BUG_ISSUES=$(gh issue list --state all --search "created:${SINCE}..${UNTIL} label:bug" \
  --limit 500 --json number -q '.[].number')

# Search commits referencing those issue numbers and extract changed files
for ISSUE_NUM in $BUG_ISSUES; do
  git log --after="$SINCE" --before="$UNTIL" --all --grep="#${ISSUE_NUM}" --name-only --format=''
done | sort | uniq -c | sort -rn | head -10
```

Note: This only works if commits reference issues with `#123` format.
If no results, mention that commit messages may not reference issue numbers.

---

### 12. Codebase Growth Trend

How is the codebase evolving? New files vs. modified files vs. deleted files.

```bash
# New files added in sprint
git log --after="$SINCE" --before="$UNTIL" --diff-filter=A --name-only --format='' | sort -u | wc -l

# Files deleted in sprint
git log --after="$SINCE" --before="$UNTIL" --diff-filter=D --name-only --format='' | sort -u | wc -l

# Files modified (not new, not deleted)
git log --after="$SINCE" --before="$UNTIL" --diff-filter=M --name-only --format='' | sort -u | wc -l

# Net LOC change
git log --after="$SINCE" --before="$UNTIL" --numstat --format='' | \
  awk '{ added += $1; removed += $2 } END { print "Net: " added - removed " lines" }'
```

---

### 13. Commit Frequency (Distribution Over Time)

Show how commits are distributed across the sprint (daily breakdown).

```bash
git log --after="$SINCE" --before="$UNTIL" --format='%ad' --date=short | \
  sort | uniq -c | sort -k2
```

Present as a simple ASCII bar chart or markdown table showing commits per day.
Highlight patterns like "sprint-end panic" (lots of commits in the last days).

---

## Charts & Diagrams

Generate visual charts for the report. Use **Mermaid** for Markdown/File output
(GitHub/GitLab render these natively) and **matplotlib PNGs** for Slack output
(uploaded as image files).

**Prerequisite for Slack charts:** `python3` with `matplotlib` must be installed.
If not available, fall back to ASCII charts and warn the user.

### Chart 1: Commit Frequency (Bar Chart)

**Mermaid (for Markdown/File):**

Include this in the report markdown under the Commit Frequency section:

````markdown
```mermaid
xychart-beta
  title "Commit Frequency"
  x-axis ["2026-02-17", "2026-02-18", ...]
  y-axis "Commits" 0 --> MAX_VALUE
  bar [5, 12, 3, ...]
```
````

Build the Mermaid block dynamically from the commit frequency data.
Use the dates as x-axis labels and commit counts as bar values.

**matplotlib (for Slack):**

```bash
python3 -c "
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from datetime import datetime

# Data: replace with actual collected values
dates = ['2026-02-17', '2026-02-18']  # from git log
counts = [5, 12]  # from git log

fig, ax = plt.subplots(figsize=(10, 4))
x = [datetime.strptime(d, '%Y-%m-%d') for d in dates]
ax.bar(x, counts, color='#4A90D9', width=0.8)
ax.set_title('Commit Frequency', fontsize=14, fontweight='bold')
ax.set_ylabel('Commits')
ax.xaxis.set_major_formatter(mdates.DateFormatter('%m-%d'))
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.savefig('/tmp/sprint-chart-commits.png', dpi=150)
print('Chart saved: /tmp/sprint-chart-commits.png')
"
```

---

### Chart 2: Top Contributors (Horizontal Bar Chart)

**Mermaid (for Markdown/File):**

````markdown
```mermaid
xychart-beta horizontal
  title "Top Contributors (Commits)"
  x-axis ["Alice", "Bob", "Carol"]
  y-axis "Commits" 0 --> MAX_VALUE
  bar [25, 18, 12]
```
````

**matplotlib (for Slack):**

```bash
python3 -c "
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

# Data: replace with actual collected values
authors = ['Alice', 'Bob', 'Carol']
commits = [25, 18, 12]
added = [1200, 800, 400]
removed = [300, 200, 100]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))

# Commits
ax1.barh(authors, commits, color='#4A90D9')
ax1.set_title('Commits per Author', fontweight='bold')
ax1.set_xlabel('Commits')
ax1.invert_yaxis()

# LOC
y_pos = range(len(authors))
ax2.barh([y - 0.15 for y in y_pos], added, height=0.3, color='#50C878', label='Added')
ax2.barh([y + 0.15 for y in y_pos], [-r for r in removed], height=0.3, color='#E74C3C', label='Removed')
ax2.set_yticks(list(y_pos))
ax2.set_yticklabels(authors)
ax2.set_title('LOC per Author', fontweight='bold')
ax2.set_xlabel('Lines of Code')
ax2.legend()
ax2.invert_yaxis()

plt.tight_layout()
plt.savefig('/tmp/sprint-chart-contributors.png', dpi=150)
print('Chart saved: /tmp/sprint-chart-contributors.png')
"
```

---

### Chart 3: LOC Trend (Line Chart)

Daily lines added vs removed over the sprint.

**Mermaid (for Markdown/File):**

````markdown
```mermaid
xychart-beta
  title "LOC Changes Over Time"
  x-axis ["02-17", "02-18", ...]
  y-axis "Lines" 0 --> MAX_VALUE
  line "Added" [120, 85, ...]
  line "Removed" [30, 45, ...]
```
````

Build from `git log --numstat` grouped by date.

**matplotlib (for Slack):**

```bash
python3 -c "
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from datetime import datetime

# Data: replace with actual collected values
dates = ['2026-02-17', '2026-02-18']
added = [120, 85]
removed = [30, 45]

fig, ax = plt.subplots(figsize=(10, 4))
x = [datetime.strptime(d, '%Y-%m-%d') for d in dates]
ax.fill_between(x, added, alpha=0.3, color='#50C878')
ax.plot(x, added, color='#50C878', linewidth=2, marker='o', label='Added')
ax.fill_between(x, removed, alpha=0.3, color='#E74C3C')
ax.plot(x, removed, color='#E74C3C', linewidth=2, marker='o', label='Removed')
ax.set_title('LOC Changes Over Time', fontsize=14, fontweight='bold')
ax.set_ylabel('Lines of Code')
ax.legend()
ax.xaxis.set_major_formatter(mdates.DateFormatter('%m-%d'))
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.savefig('/tmp/sprint-chart-loc-trend.png', dpi=150)
print('Chart saved: /tmp/sprint-chart-loc-trend.png')
"
```

Collect LOC per day with:
```bash
# NOTE: --format='DATE:%ad' puts the date as part of the field, so use substr() to strip the prefix
git log --after="$SINCE" --before="$UNTIL" --numstat --format='DATE:%ad' --date=short | \
  awk '/^DATE:/ { date=substr($0,6); next }
       NF==3 && $1 != "-" { added[date]+=$1; removed[date]+=$2 }
       END { for (d in added) print d, "+"added[d], "-"removed[d] }' | sort
```

---

### Chart 4: Hotspot / Churn Scatter Plot

Plots files by change frequency (x-axis) vs churn (y-axis). Files in the
top-right quadrant are priority refactoring targets.

**Mermaid:** Mermaid does not support scatter plots. For Markdown/File output,
use a table with visual indicators instead:

````markdown
| File | Changes | Churn | Priority |
|------|---------|-------|----------|
| foo.ts | 8 | 245 | REFACTOR |
| bar.ts | 6 | 180 | REFACTOR |
| baz.ts | 5 | 30 | OK |
````

Mark files as `REFACTOR` if they appear in both hotspots top-10 AND churn top-10.

**matplotlib (for Slack):**

```bash
python3 -c "
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt

# Data: replace with actual collected values
files =      ['foo.ts',  'bar.ts',  'baz.ts']
changes =    [8,         6,         5]          # change frequency
churn =      [245,       180,       30]         # total churn (added + removed)
is_hotspot = [True,      True,      False]

fig, ax = plt.subplots(figsize=(10, 6))
colors = ['#E74C3C' if h else '#4A90D9' for h in is_hotspot]
ax.scatter(changes, churn, c=colors, s=100, alpha=0.7, edgecolors='black')

for i, f in enumerate(files):
    ax.annotate(f.split('/')[-1], (changes[i], churn[i]),
                textcoords='offset points', xytext=(5, 5), fontsize=8)

ax.set_title('Hotspot / Churn Analysis', fontsize=14, fontweight='bold')
ax.set_xlabel('Change Frequency (# commits)')
ax.set_ylabel('Code Churn (LOC added + removed)')
ax.axhline(y=sum(churn)/len(churn), color='gray', linestyle='--', alpha=0.5, label='Avg churn')
ax.axvline(x=sum(changes)/len(changes), color='gray', linestyle='--', alpha=0.5, label='Avg changes')
ax.legend(['Refactoring target', 'Normal', 'Avg threshold'], loc='upper left')
plt.tight_layout()
plt.savefig('/tmp/sprint-chart-hotspots.png', dpi=150)
print('Chart saved: /tmp/sprint-chart-hotspots.png')
"
```

---

### Uploading Chart PNGs to Slack

After generating the chart PNGs, upload each one using the file upload workflow
from the **slack-notify** skill. Upload all charts in sequence, then send the
.md report file last. Use a descriptive `initial_comment` for each chart or
batch them in a single thread.

```bash
# Upload each chart PNG — repeat for each chart file
# IMPORTANT: export env vars so they're available to subprocesses
set -a
[[ -f ~/.config/opencode/.env ]] && source ~/.config/opencode/.env
[[ -f .env.local ]] && source .env.local
set +a

for CHART in /tmp/sprint-chart-*.png; do
  FILE_NAME=$(basename "$CHART")
  FILE_SIZE=$(stat -f%z "$CHART" 2>/dev/null || wc -c < "$CHART" | tr -d ' ')

  URL_RESPONSE=$(curl -s -X POST "https://slack.com/api/files.getUploadURLExternal" \
    -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
    -H "Content-Type: application/x-www-form-urlencoded" \
    --data-urlencode "filename=${FILE_NAME}" \
    --data-urlencode "length=${FILE_SIZE}")

  UPLOAD_URL=$(echo "$URL_RESPONSE" | jq -r '.upload_url')
  FILE_ID=$(echo "$URL_RESPONSE" | jq -r '.file_id')

  curl -s -X POST "$UPLOAD_URL" -F "file=@${CHART}" > /dev/null

  COMPLETE_PAYLOAD=$(jq -n \
    --arg file_id "$FILE_ID" \
    --arg channel "$SLACK_DEFAULT_CHANNEL" \
    '{files: [{id: $file_id}], channel_id: $channel}')

  curl -s -X POST "https://slack.com/api/files.completeUploadExternal" \
    -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "$COMPLETE_PAYLOAD" > /dev/null
done

# Clean up temp files
rm -f /tmp/sprint-chart-*.png
```

---

## Report Template

Assemble the collected data into this structure:

```markdown
# Sprint Report: [REPO] — [SPRINT_LABEL]
**Period:** [SINCE] to [UNTIL]

---

## Summary
| Metric | Value |
|--------|-------|
| Total Commits | X |
| Unique Contributors | Y |
| Issues Created | Z |
| Issues Closed | W |
| Issues Still Open | V |
| Lines Added | +A |
| Lines Removed | -B |
| Net LOC Change | +/-N |
| New Files | F |
| Deleted Files | D |
| Avg Issue Cycle Time | T days |

---

## Top Contributors
| Author | Commits | Lines Added | Lines Removed |
|--------|---------|-------------|---------------|
| ... | ... | ... | ... |

---

## Commit Frequency
| Date | Commits | |
|------|---------|---|
| 2026-02-17 | 5 | ##### |
| 2026-02-18 | 12 | ############ |
| ... | ... | ... |

---

## Hotspots (Most Changed Files)
| # Changes | File |
|-----------|------|
| ... | ... |

---

## Refactoring Candidates (High Churn)
Files with high code churn — frequently rewritten code that may benefit from refactoring.

| Churn (LOC) | Added | Removed | File | Also Hotspot? |
|-------------|-------|---------|------|---------------|
| ... | ... | ... | ... | Yes/No |

> Files marked "Also Hotspot? Yes" are **priority refactoring targets**.

---

## Knowledge Distribution / Bus Factor
| File | # Authors | Authors | Risk |
|------|-----------|---------|------|
| ... | 1 | Alice | SILO |
| ... | 3 | Alice, Bob, Carol | OK |

> Files with only 1 author are **knowledge silos** — consider pairing or code review.

---

## Temporal Coupling (Hidden Dependencies)
| Co-Changes | File A | File B | Likely Reason |
|------------|--------|--------|---------------|
| ... | ... | ... | (test+impl / tight coupling / unknown) |

> Pairs that are NOT test+implementation may indicate **tight coupling** worth refactoring.

---

## Issue Breakdown

### By Label
| Label | Count |
|-------|-------|
| bug | X |
| feature | Y |
| ... | ... |

### Longest Cycle Times
| Issue | Title | Cycle Time |
|-------|-------|------------|
| #123 | ... | X days |

---

## Bug Hotspots
Files most frequently associated with bug-fix commits.

| # Bug Refs | File |
|------------|------|
| ... | ... |

> These files may need extra attention — tests, reviews, or refactoring.

---

## Codebase Growth
| Metric | Value |
|--------|-------|
| New Files | +F |
| Deleted Files | -D |
| Modified Files | M |
| Net LOC Change | +/-N |

---

*Report generated by Josie — your friendly neighborhood code analyst.*
```

---

## Output Handling

The user may select **one or more** output formats. Generate the report content once,
then deliver it to each selected format in sequence. The report markdown and chart data
are reused across all outputs — no need to re-collect metrics.

### Terminal (default)
Render the report directly as markdown in the chat. This is the default if the
user doesn't specify an output format.

### Slack
After generating the report, use the **slack-notify** skill to send it in two steps:

1. **Upload the full report as a `.md` file** using the file upload workflow from
   slack-notify (3-step upload: get URL, upload content, complete with channel).
   Save the report to a temp file first, then upload it:

```bash
set -a
[[ -f ~/.config/opencode/.env ]] && source ~/.config/opencode/.env
[[ -f .env.local ]] && source .env.local
set +a

# Write full report to temp file
REPORT_FILE=$(mktemp /tmp/sprint-report-XXXXXX.md)
cat > "$REPORT_FILE" << 'REPORT_EOF'
... full markdown report content here ...
REPORT_EOF

FILE_NAME="sprint-report-$(date +%Y-%m-%d).md"
FILE_SIZE=$(wc -c < "$REPORT_FILE" | tr -d ' ')

# Step 1: Get upload URL
URL_RESPONSE=$(curl -s -X GET "https://slack.com/api/files.getUploadURLExternal" \
  -H "Authorization: Bearer ${SLACK_BOT_TOKEN}" \
  -G \
  --data-urlencode "filename=${FILE_NAME}" \
  --data-urlencode "length=${FILE_SIZE}")

UPLOAD_URL=$(echo "$URL_RESPONSE" | jq -r '.upload_url')
FILE_ID=$(echo "$URL_RESPONSE" | jq -r '.file_id')

# Step 2: Upload file content
curl -s -X POST "$UPLOAD_URL" -F "file=@${REPORT_FILE}"

# Step 3: Complete upload and share to channel with summary as comment
CHANNEL_ID="$SLACK_DEFAULT_CHANNEL"  # or $SLACK_USER_ME etc.
```

2. **Send a condensed summary as the `initial_comment`** on the file upload, containing:
   - Summary numbers (commits, issues, LOC)
   - Top 3 contributors
   - Top 3 hotspots
   - Top 3 refactoring candidates
   - Any knowledge silos found

**IMPORTANT:** For multiline `initial_comment` text, you MUST use python3 with `json.dumps()`
to build the JSON payload. The `jq --arg` approach BREAKS on literal newlines in shell strings.

```bash
# Use python3 for the completeUploadExternal call with multiline initial_comment
python3 << PYEOF
import urllib.request, json, os

token = os.environ["SLACK_BOT_TOKEN"]
channel = os.environ["SLACK_DEFAULT_CHANNEL"]

summary = """*Sprint Report: [REPO]*
*Period:* $SINCE to $UNTIL

*Key Metrics:*
... condensed summary here ...

Full report attached."""

data = json.dumps({
    "files": [{"id": "$FILE_ID", "title": "Sprint Report"}],
    "channel_id": channel,
    "initial_comment": summary
}).encode()

req = urllib.request.Request(
    "https://slack.com/api/files.completeUploadExternal",
    data=data,
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
)
resp = urllib.request.urlopen(req)
result = json.loads(resp.read().decode())
print("OK" if result.get("ok") else f"ERROR: {result}")
PYEOF

rm -f "$REPORT_FILE"
```

### File
Save the full report as a markdown file:
```bash
REPORT_FILE="sprint-report-$(date +%Y-%m-%d).md"
# Write the report content to $REPORT_FILE
```
Confirm the file path to the user after saving.

## Workflow Summary

1. **Ask** for time range, repo (optional), output format(s) — **multiple allowed**
2. **Resolve** dates to ISO format, determine repo (local or remote)
3. **Collect** all metrics using the bash commands above
4. **Assemble** the report using the template (generate once, reuse for all outputs)
5. **Output** to each selected format in sequence:
   - `terminal` → render markdown in chat
   - `slack` → upload charts + .md file with condensed summary
   - `file` → save as `sprint-report-YYYY-MM-DD.md`
6. **Offer** to drill deeper into any section if the user wants

## Error Handling

- If `gh` is not authenticated: tell user to run `gh auth login`
- If no commits in range: warn that the date range may be wrong
- If bug hotspots return empty: note that commits may not reference issues
- If temporal coupling returns nothing: the sprint may be too short or commits too small
- Always show partial results even if some metrics fail

## When to Use

Trigger this skill when the user asks for:
- Sprint reports, sprint summaries, sprint statistics
- Repository analysis for a time period
- Iteration reviews or retrospective data
- Code health or codebase analysis over time
