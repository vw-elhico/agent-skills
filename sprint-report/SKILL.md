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

3. **Output format** — ask if not specified:
   - `terminal` (default) — render report as markdown in chat
   - `slack` — send via slack-notify skill
   - `file` — save as `sprint-report-YYYY-MM-DD.md` in repo root

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
# NOTE: use --pretty=format: to avoid commit body leaking into output
HOTSPOT_FILES=$(git log --after="$SINCE" --before="$UNTIL" --name-only --pretty=format:'' | \
  grep -v '^$' | sort | uniq -c | sort -rn | head -10)

echo "$HOTSPOT_FILES" | while read -r COUNT FILE; do
  [ -z "$FILE" ] && continue
  AUTHORS_COUNT=$(git log --after="$SINCE" --before="$UNTIL" -- "$FILE" --pretty=format:'%aN' | sort -u | grep -c .)
  NAMES=$(git log --after="$SINCE" --before="$UNTIL" -- "$FILE" --pretty=format:'%aN' | sort -u | paste -sd', ' -)
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

### Terminal (default)
Render the report directly as markdown in the chat. This is the default if the
user doesn't specify an output format.

### Slack
After generating the report, use the **slack-notify** skill to send it.
Since Slack messages have length limits, send a **condensed summary** via Slack:
- Summary table (commits, issues, LOC)
- Top 3 contributors
- Top 3 hotspots
- Top 3 refactoring candidates
- Any knowledge silos found

Load the slack-notify skill and follow its workflow for sending.
**IMPORTANT:** Always use the python3 method from slack-notify for multiline messages.
The `jq --arg` approach breaks on newlines in shell strings.

### File
Save the full report as a markdown file:
```bash
REPORT_FILE="sprint-report-$(date +%Y-%m-%d).md"
# Write the report content to $REPORT_FILE
```
Confirm the file path to the user after saving.

## Workflow Summary

1. **Ask** for time range, repo (optional), output format (optional)
2. **Resolve** dates to ISO format, determine repo (local or remote)
3. **Collect** all metrics using the bash commands above
4. **Assemble** the report using the template
5. **Output** in the requested format (terminal / slack / file)
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
