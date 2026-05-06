---
description: Reconstruct a daily ticket-centric work timeline from Jira/GitHub, git, calendar, and Slack signals
disable-model-invocation: true
argument-hint: "[YYYY-MM-DD]"
---

You are tasked with reconstructing the user's work for a single day as a ticket-centric timeline suitable for time-tracking entry (e.g. Toggl). You combine evidence from calendar, git history, pull requests, issue trackers, and Slack into a non-overlapping timeline.

## Output discipline (applies throughout this skill)

The user wants the **final timeline**, not a play-by-play of how you built it. Keep all reasoning, scratch work, timezone arithmetic, per-block deliberation, rounding mechanics, and self-check output **internal** — do not print it.

**Do not emit any of these to the user:**
- Narration like "Now constructing the timeline." / "Let me lay out the rounded timeline" / "Re-converting UTC offsets..."
- UTC↔local conversion tables, per-PR/commit timestamp dumps, or "Wait — let me reconsider" passages
- Pre-rounding draft timelines followed by a rounded version (only print the final one)
- "Per-ticket blocks" / "Section 6 rounding" / "Self-check (6a)" headers showing your process
- Restating signals you already gathered before rendering the table
- "Total: X matches workday ✓" lines — keep self-check internal; only surface drift if it fails

**Do emit, in this order, and nothing else:**
1. A **single short status line** before signal gathering (e.g. `Gathering signals for 2026-05-06…`) — one line, not a section
2. A **single short status line** if Section 2.0 found a Notion daily log (e.g. `Found daily log for 2026-05-06 — using as ground truth.`) or didn't (`No Notion daily log for 2026-05-06; reconstructing from signals.`)
3. The **final rendered output** from Section 6: the heading and the table only — no Evidence section, no Caveats block, nothing between or around them
4. The single trailing prompt: `Apply edits, accept as-is, or regenerate?`

Evidence is gathered and used internally to anchor each row, but **never printed**. If you catch yourself writing a sentence that explains how you arrived at a row, delete it — it stays in your head.

Caveats are also internal-only. If a hard blocker occurred (identity preflight failed, MCP unavailable, no signals at all), surface it as a question or error *before* attempting to render the timeline — never as a footer on a rendered table.

## Clarifying-question style (applies throughout this skill)

When you need information from the user — config gaps, identity disambiguation, ticket-creation approval, timeline edits, write confirmations — **ask one question at a time**. Never emit a wall of text containing multiple questions the user has to answer all at once.

Preferred forms, in order:
1. **A single `AskUserQuestion` widget** with a focused prompt and a small set of options (or free-text). One concept per call.
2. **A short numbered/multiple-choice prompt** in plain text when no widget tool is available — but still only one decision per turn.
3. Free-text prompt — last resort, and still scoped to one decision.

If you have several unresolved questions, queue them and ask sequentially, using the previous answer to inform the next. Do not batch them into a single message like "Also, please confirm: (a) … (b) … (c) …". Batching is the failure mode this rule exists to prevent.

The only exceptions are **review-and-approve batches** that are inherently a single decision over many items — e.g. "approve / edit / skip" on a list of proposed new tickets (Section 3), or "yes / edit / abort" on the Toggl write batch (Section 7c). Those are one decision presented over a structured list, not multiple independent questions; render them as a table or numbered list with a single trailing prompt.

**Command accepts an optional date**: `/toggl-tamer [YYYY-MM-DD]`
- If no date is provided, default to **today** in the user's local timezone.
- Accept also `yesterday`, `today`, or a weekday name; resolve to an absolute date before proceeding.

The user's email in `~/.claude/CLAUDE.md` under `# userEmail` is **only one of several identities** — it is typically the work email used for calendar/Atlassian, but the user's git author email and Slack user ID are often *different* and must be discovered separately. The skill must resolve and cache all of them in config (see Section 0d). Treating `# userEmail` as the universal filter is a known silent-failure mode: the wrong identity returns zero results, and the skill confidently reports "you did nothing today".

---

## 0. Load Configuration

Configuration lives at `~/.claude/skills/toggl-tamer/config.json`. Schema:

```json
{
  "projects": [
    {
      "name": "Acme Web",
      "tracker": "jira",
      "jiraProjectKey": "ACME",
      "atlassianSiteUrl": "https://labrys.atlassian.net",
      "repos": ["/Users/barryearsman/projects/acme-web"],
      "githubRepoSlug": "labrys/acme-web",
      "togglProjectName": "Acme Web"
    },
    {
      "name": "Internal Tools",
      "tracker": "github",
      "repos": ["/Users/barryearsman/projects/internal-tools"],
      "githubRepoSlug": "labrys/internal-tools",
      "togglProjectName": "Internal Tools"
    }
  ],
  "workdayDefaults": {
    "startTime": "09:00",
    "endTime": "17:30",
    "lunchMinutes": 60,
    "lunchAroundTime": "12:30",
    "timezone": "Australia/Brisbane"
  },
  "userIdentities": {
    "primaryEmail": "barry@labrys.io",
    "gitAuthorEmails": ["barry@earsman.com", "barry@labrys.io"],
    "atlassianAccountId": "712020:...",
    "notionUserId": "...",
    "slackUserId": "U07V1KQMVLH"
  },
  "notionDailyLog": {
    "enabled": true,
    "titlePattern": "^\\d{4}-\\d{2}-\\d{2}$",
    "parentPageId": null
  },
  "internalProject": {
    "name": "Internal / Ops",
    "label": "(internal)",
    "togglProjectName": "Internal / Ops"
  },
  "slackEnabled": true,
  "calendarEnabled": true,
  "notionEnabled": true,
  "togglEnabled": true,
  "togglWrite": {
    "skipMeta": ["(lunch)", "[unaccounted]"],
    "tagWithTicket": true
  }
}
```

**Critical identity fields** (the "silent failure" risks):
- `userIdentities.gitAuthorEmails` — list of every email that has authored commits in any configured repo. The user's `# userEmail` is often *not* the git author email (e.g. `barry@labrys.io` vs `barry@earsman.com`). Filtering `git log` by the wrong identity returns zero results and the skill will confidently report "no commits today".
- `userIdentities.slackUserId` — Slack's `from:@me` modifier silently returns zero results in many workspaces. You **must** use `from:<@U…>` form with the resolved user ID.
- `notionDailyLog` — if enabled, the skill **reads the most recent daily-log page first** as ground truth (Section 2.0) before stitching together other signals.
- `internalProject` — pseudo-project for non-project work: meetings without a project ticket, process discussions, ticket triage, code review on others' PRs, tooling. Without this, internal time gets dropped from the timeline.

If the config file does **not exist**, **derive a draft project list automatically** before asking the user, then present it for confirmation. Do **not** invent projects — only include items backed by evidence from the sources below.

### 0a. Auto-discover projects

Run these probes in parallel:

**GitHub (via `gh`):**
```bash
# Repos the user has pushed to in the last 90 days
gh api graphql -f query='
  query { viewer {
    contributionsCollection {
      commitContributionsByRepository(maxRepositories: 50) {
        repository { nameWithOwner url defaultBranchRef { name } }
        contributions { totalCount }
      }
    }
  }}' --jq '.data.viewer.contributionsCollection.commitContributionsByRepository[] | {repo: .repository.nameWithOwner, commits: .contributions.totalCount}'

# PRs the user authored in the last 90 days (catches repos missed above)
gh search prs --author=@me --created=">$(date -v-90d +%F)" --json repository --jq '[.[].repository.nameWithOwner] | unique'
```

**Jira (via Atlassian MCP):**
- Call `getAccessibleAtlassianResources` to list site URLs.
- For each site, call `getVisibleJiraProjects` and keep projects where the user has activity in the last 90 days, found via `searchJiraIssuesUsingJql`:
  `assignee = currentUser() OR reporter = currentUser() OR comment ~ currentUser() AND updated >= -90d`. Group results by `project.key` and keep projects with ≥ 1 hit.

**Local repos:**
- Scan `~/projects` (and `~/code`, `~/dev`, `~/src` if present) one level deep for directories containing `.git`. Use `git -C <dir> remote get-url origin` to map each local path to a GitHub repo slug.
- For each repo, also harvest Jira project keys directly from commit history — this is a **first-class discovery signal**, not just a pairing signal:
  ```bash
  git -C <repo> log --since=-90d --pretty='%s %D' | grep -oE '[A-Z]{2,}-[0-9]+' | awk -F- '{print $1}' | sort | uniq -c | sort -rn
  ```
  Any project key that appears ≥ 3 times in the last 90 days of commits/branches is a candidate Jira project — propose it even if there's **no Jira activity for the user** (assignee/reporter/comment) on that project. Users who never move tickets in Jira but commit constantly are otherwise invisible to JQL-based discovery.

### 0b. Build the draft

Combine the discoveries:
- Each GitHub repo with recent activity becomes a candidate project. Pair it with the most frequent Jira project key from the commit-message harvest in 0a — even if that Jira project had no JQL hits for the user.
- Jira projects with activity but no matching GitHub repo become tracker-only candidates (`tracker: "jira"`, no `repos`).
- Jira project keys that appeared **only** via the commit-message harvest (no JQL activity for the user, no obviously matching repo) are still candidates — list them with a note `(discovered via commit messages)` so the user can confirm they're real and supply the site URL.
- Local repos that match a GitHub candidate get their path attached as `repos[]`. A local repo with no GitHub match is still listed (with `githubRepoSlug: null`) so the user can decide.

Name each candidate after the GitHub repo (e.g. `labrys/acme-web`) or the Jira project name when there's no repo. Sort by recent activity volume, descending.

### 0c. Confirm with the user

Present the draft as a numbered list, e.g.:

```
I found these candidate projects from the last 90 days:

  1. labrys/acme-web         (jira: ACME, 47 commits, 12 PRs, local: ~/projects/acme-web)
  2. labrys/internal-tools   (github issues, 8 PRs, local: ~/projects/internal-tools)
  3. PLAT (Jira only)        (3 issues touched, no matching repo)

Reply with: keep numbers (e.g. "1,2"), edit details, or "all" to keep everything.
Add anything missing? Anything to drop?
```

Iterate until the user confirms. Then ask only for the **gaps** that auto-discovery couldn't fill — **one question at a time**, per the clarifying-question style above. Do not concatenate the gap list into a single multi-part prompt; resolve each before moving to the next, so the answer to one can simplify or skip the rest.

Gap order:
1. For confirmed Jira projects without a known site URL → ask for the Atlassian site URL (one project per question if there are several)
2. For confirmed projects without a local repo path → ask for the path (or skip)
3. Whether to use Slack, Google Calendar, and Notion signals — ask per source, defaulting to yes if the MCP server is available
4. Workday defaults — present the suggested defaults (09:00 / 17:30 / 60min lunch / 12:30 lunch midpoint) as a single accept-or-edit decision, not four separate questions

Write the config and confirm before proceeding. On subsequent runs, read it without prompting unless the user passes `--reconfigure`.

If auto-discovery returns **nothing** (no GitHub access, no Atlassian MCP, no local repos), fall back to fully manual entry — but tell the user *why* you're falling back.

---

## 0d. Identity Resolution Preflight (silent-failure prevention)

The `# userEmail` from CLAUDE.md is **only one identity**. Before any signal gathering, resolve and cache every identity that filters can use. Any of these resolving to the wrong value causes silent zero-result queries that look like "the user did no work today" — the highest blast-radius failure mode for this skill.

For each identity below: if it's missing or empty in `config.json` under `userIdentities`, **resolve it now and write it back** before continuing.

### Git author emails (`gitAuthorEmails: string[]`)

The user's git author email is often *different* from their work email — e.g. a personal address used for commits while `# userEmail` is the work address. `git log --author=<wrong-email>` returns zero rows silently.

For each repo across all configured projects, run:
```bash
git -C <repo> log --since=-90d --pretty='%ae' | sort -u
```
Aggregate the union, drop bot/CI addresses (`*[bot]@*`, `noreply@github.com`, etc.), and present every remaining candidate to the user:
```
I see commits in your configured repos from these author emails:
  1. barry@earsman.com   (412 commits, last 2026-05-06)
  2. barry@labrys.io     (38 commits, last 2026-04-02)
  3. baz@example.com     (1 commit, 2026-01-15)
Which of these are you? (e.g. "1,2")
```
Save the chosen list as `userIdentities.gitAuthorEmails`. **Always use multiple `--author` flags** when filtering commits — one per email — so a user with mismatched identities is covered:
```bash
git -C <repo> log --author="barry@earsman.com" --author="barry@labrys.io" --since=... --until=...
```

### Slack user ID (`slackUserId: string`)

Slack's `from:@me` modifier silently returns zero results in many workspaces. **Never** use `from:@me`. Always use `from:<@U…>` form with the resolved user ID.

If `slackUserId` is unset, resolve it once via `slack_search_users` against the user's name or email and cache it. If multiple users match, ask the user to disambiguate. Use the cached ID for *every* Slack search going forward.

### Notion user ID (`notionUserId: string`)

Already documented in Section 2f. Resolve via `notion-get-users` and cache.

### Atlassian account ID (`atlassianAccountId: string`)

Some JQL filters require the account ID (`assignee = "<accountId>"`) rather than `currentUser()`. Resolve via `atlassianUserInfo` and cache.

### Hard rule

If any identity used by an enabled signal source can't be resolved, **stop** and prompt the user — do not fall back to `currentUser()` / `from:@me` / `userEmail` and produce a confidently-wrong empty timeline. The cost of asking is one prompt; the cost of a silent miss is the user re-doing the day's work by hand.

---

## 0e. MCP Availability Preflight

Before establishing the day window, verify that the integrations needed for the configured projects and signal flags are actually available. **If required integrations are missing, stop and prompt the user — do not proceed with a partial signal set.**

### Required integrations

A signal is **required** if its corresponding flag is true in config OR if at least one configured project depends on it:

| Integration | Required when... | Probe |
|-------------|------------------|-------|
| `gh` CLI authed | Any project has `githubRepoSlug` | `gh auth status` exits 0 |
| Atlassian MCP | Any project has `tracker: "jira"` | Tool `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` is callable AND returns ≥ 1 resource |
| Google Calendar MCP | `calendarEnabled: true` | Tool `mcp__claude_ai_Google_Calendar__authenticate` is present; a probe call succeeds (not in `authenticate` state) |
| Slack MCP | `slackEnabled: true` | Tool `mcp__claude_ai_Slack__slack_search_public_and_private` is callable on a trivial query |
| Notion MCP | `notionEnabled: true` | Tool `mcp__claude_ai_Notion__notion-search` is callable AND `notionUserId` resolvable via `notion-get-users` |
| Toggl MCP | `togglEnabled: true` | Tool `mcp__toggl__get_workspaces` returns ≥ 1 workspace AND `mcp__toggl__create_time_entry` is present in the tool list (older builds of the MCP only expose `start_timer` and cannot back-fill — flag and stop) AND every project's `togglProjectName` (including `internalProject.togglProjectName`) is found in `mcp__toggl__get_projects` |
| Local git | Any project has `repos[]` | `git -C <repo> rev-parse --git-dir` exits 0 for each |

Run all probes in parallel. Treat a probe that times out, errors, or requires re-auth as **missing**.

### When something is missing

Build a single status block, e.g.:

```
Toggl Tamer preflight — 2026-05-06

Required integrations:
  ✓ gh CLI authed (labrys-Group)
  ✓ Atlassian MCP (labrys.atlassian.net)
  ✗ Google Calendar MCP — not authenticated
      Run: use the mcp__claude_ai_Google_Calendar__authenticate tool, or disable
      calendar signals with `calendarEnabled: false` in config.
  ✓ Slack MCP
  ✗ Notion MCP — tool not available in this session
      Add the Notion MCP server, or disable with `notionEnabled: false` in config.
  ✓ Local git for ~/projects/labrys-website-v2
  ✓ Toggl MCP (workspace: Labrys, create_time_entry available)
```

Then **stop and prompt** with three options:

1. **Fix and re-run** — user resolves the missing integrations, then re-invokes `/toggl-tamer`.
2. **Disable and continue** — user accepts running with reduced signals; update the relevant config flag (`calendarEnabled`, `slackEnabled`, `notionEnabled`) to `false` and proceed. Surface this as a caveat in the final output.
3. **Abort** — exit cleanly without producing a timeline.

Do **not** silently proceed with missing integrations and bury the gap in a "Caveats" section — that produced the previous trial run where calendar was missing but the timeline ran anyway. The user must explicitly choose option 2 to continue with reduced signals.

### Hard requirements (no fallback allowed)

The following gaps **block execution** even if the user chooses "disable and continue". Prompt for fix or abort:

- No working git access for any configured project's repos → cannot derive commit signals → can't build a timeline.
- No Jira AND no GitHub access for any configured project → no way to associate or create tickets → output would be PR-centric, which Section 3 forbids.
- The user's email (`# userEmail` in CLAUDE.md) is missing → cannot filter signals by author.
- `togglEnabled: true` but Toggl MCP is missing OR `create_time_entry` is unavailable OR a configured `togglProjectName` is not found in Toggl → cannot complete Section 7. Either rebuild the MCP server with `create_time_entry`, fix the project-name mismatch, or set `togglEnabled: false` (which downgrades the run to "preview only" and prints a caveat instead of writing).

For these, the only options are **Fix and re-run** or **Abort** — do not offer "disable and continue".

---

## 1. Establish the Day Window

Compute `dayStart` and `dayEnd` as ISO 8601 timestamps in the configured timezone covering 00:00:00 → 23:59:59 of the target date. Use these for ALL queries. Do not use UTC unless that matches the configured timezone.

---

## 2. Gather Signals (run in parallel where possible)

### 2.0. Notion daily-log (run FIRST, before everything else)

If `notionDailyLog.enabled` is true, **read the most recent daily-log page in Notion before doing anything else**. A user-curated daily log is far more accurate than any reconstruction from 30 separate signals — it is ground truth for what the user did and roughly when. The point of this skill is *not* to ignore that and rebuild it from scratch.

How to find it:
1. Use `notion-search` filtered to `last_edited_by: notionUserId`, sorted by `last_edited_time` desc.
2. Scan results for pages whose **title matches `notionDailyLog.titlePattern`** (default: a date like `2026-05-06`) — or whose title is "today's date", "yesterday's date", or a rolling header.
3. If `notionDailyLog.parentPageId` is set, restrict to children of that page.
4. Pick the page whose title or content matches the **target date** of this run, falling back to the most recently edited matching page.

Once found:
- `notion-fetch` the page and parse it into `{time?, summary, ticketHints[]}` entries.
- Treat ticket-key mentions in the daily log as **strong association evidence** for any commits/PRs/edits in the same time range.
- Treat free-text entries (e.g. "52min — Toggl automation discussion with Joshua") as evidence for `(internal)` blocks (Section 3a) when no project ticket fits.
- Surface the daily log up front: "Found daily log for 2026-05-06 with 7 entries — using as ground truth and cross-checking against other signals."

If no daily log exists for the target date, fall back to the multi-signal stitching below — but tell the user: "No Notion daily log found for 2026-05-06; reconstructing from commits/PRs/calendar/Slack."

### 2a. Calendar
If `calendarEnabled`, list all events for the day where the user attended (not declined). Capture: title, start, end, attendees, description. Treat these as **fixed** timeline anchors.

### 2b. Git commits (per project, per repo)
For each repo path, filter by **every** identity in `userIdentities.gitAuthorEmails` — never just `# userEmail`. `git log` ANDs `--author` flags by repetition; pass one per email:
```bash
git -C <repo> log \
    $(printf -- '--author=%s ' "${gitAuthorEmails[@]}") \
    --since="<dayStart>" --until="<dayEnd>" \
    --pretty=format:'%H%x09%aI%x09%s%x09%D' --no-merges
```
If a single repo returns zero commits but the day has signals from other sources, log a caveat — don't silently accept the empty result. (Most likely cause: an author email used in this repo isn't in `gitAuthorEmails` yet.)
Also capture the branch each commit landed on (use `--source` or check current branch). Capture file mtimes for files touched in each commit:
```bash
git -C <repo> show --name-only --pretty=format: <sha>
```

### 2c. Pull Requests
For each `githubRepoSlug`:
```bash
gh pr list --repo <slug> --author "@me" --state all \
   --search "created:<date> OR updated:<date>" \
   --json number,title,body,createdAt,updatedAt,mergedAt,headRefName,url
```
Include PRs **created**, **updated** (with the user's commits/comments), or **merged** that day.

### 2d. Issue trackers
- **Jira projects** (via Atlassian MCP): `searchJiraIssuesUsingJql` with JQL like:
  `project = ACME AND (assignee = currentUser() OR comment ~ currentUser()) AND updated >= "<date>"`. Capture status changes, comments, assignment changes during the day window.
- **GitHub projects**: `gh issue list --repo <slug> --search "involves:@me updated:<date>"` and fetch each issue's timeline events for the day.

### 2e. Slack (optional)
If `slackEnabled`, search the user's messages for the day. **Never use `from:@me`** — Slack's `@me` modifier silently returns zero results in many workspaces, which the skill cannot distinguish from "the user said nothing today". Always use `from:<@U…>` with the cached `userIdentities.slackUserId` (resolved in Section 0d).

Run two searches in parallel and merge:
1. **All channels and DMs**: `slack_search_public_and_private` with `from:<@SLACK_USER_ID> after:<dayStart-1> before:<dayEnd+1>`.
2. **Self-DM (notes-to-self)**: `slack_search_public_and_private` with `from:<@SLACK_USER_ID> to:<@SLACK_USER_ID>`. Self-DMs often carry the most candid work-log signal — todo lists, "doing X next", links to PRs being reviewed — and are easy to miss without an explicit query.

For each match capture timestamp + channel + a short text excerpt (first 120 chars). De-dupe by message ts.

### 2f. Notion pages (optional)
If `notionEnabled`, find Notion pages the user **edited** during the day window:
- Use `notion-search` with `query_type: "internal"`, sorted by `last_edited_time` descending, then filter results where `last_edited_time` falls within `dayStart`–`dayEnd` AND `last_edited_by` matches the user (match on email or Notion user ID — resolve once and cache in config as `notionUserId`).
- For each matching page, capture: page title, URL, `last_edited_time`, parent workspace/database, and a short excerpt (first 200 chars of content via `notion-fetch` only if needed for summarisation).
- Treat Notion edits like commits: they are **end-of-work** signals, not start signals. A page edited at 14:32 means the user was working on it *before* 14:32, not starting then.

Associate Notion pages to tickets using the same rules as commits (Section 3): ticket key in title or content, otherwise group by parent database/workspace and offer to create a tracker ticket if no association is found. A standalone Notion page (e.g. a meeting note or spec) with no ticket association is allowed — surface it in the timeline as `(notion: <page title>)` rather than forcing a synthetic ticket.

### 2g. File modification timestamps (use cautiously)
For each file touched in the day's commits, capture the filesystem mtime if the file still exists locally (`stat -f %m <path>` on macOS). This *can* help bound when work *started* on a commit — but mtimes are unreliable on macOS and easy to misread.

**Sanity check before using mtimes as evidence:**
1. If all candidate file mtimes cluster within a ±5 minute window that is **not on the target day**, treat them as junk and drop them entirely. They're almost certainly the result of Spotlight indexing, format-on-open, or `git checkout` rewriting timestamps wholesale — not actual editing activity.
2. If the cluster falls on the target day but is suspiciously tight (10+ files within ±2 min), be wary: format-on-save can rewrite many files at once. Use it as a weak signal only, not a primary anchor for `start`.
3. If a single file's mtime is on the target day but well outside any commit window, prefer the commit timestamp — the mtime might be from an editor save that wasn't committed.

When mtimes are dropped or downgraded by these checks, note the assumption in evidence (e.g. `(mtime cluster discarded — likely format-on-open)`) so the user sees that we considered the signal and rejected it.

### 2h. Previous day's last ticket (lightweight)
For the project with the **most signal volume today**, find the single latest ticket the user touched on the previous working day with any signal — commit, PR activity, Jira state change, Notion edit. Skip weekends and gap days, cap lookback at 7 days. Record `{ticket, lastSignalAt}` as `previousDayLastTicket`.

This is used in Section 4a only as a one-line prompt to the user — not as the basis for an automatically-inserted block.

---

## 3. Associate Commits & PRs to Tickets

For each commit and PR, attempt to associate it to a ticket using these signals **in priority order**, stopping at the first match:

1. **Explicit ticket key** in commit message, branch name, or PR title/body matching the configured Jira project key pattern (e.g. `ACME-123`) or `#123` for GitHub issues in that repo.
2. **Branch name** containing a ticket key (e.g. `feature/ACME-123-add-login`).
3. **PR linkage** — if the commit's SHA appears in a PR, inherit that PR's ticket association.
4. **Body references** — Jira/GitHub auto-link patterns in the PR body (`Closes #45`, `Fixes ACME-123`).

If **no ticket** can be associated:
1. Group orphan commits/PRs by branch name + repo. Multiple PRs on the same branch belong to the same group. PRs that share a branch prefix (`feature/lcp-perf-round-1` and `feature/lcp-perf-round-2`) or that touch overlapping files within a 4-hour window belong to the same group.
2. Summarise the work (use the commit messages and changed paths) into a 1-sentence title.
3. **Create a ticket** in the appropriate tracker. This step is **mandatory**, not optional — you must propose a ticket for every orphan group. Use the three-call sequence below for Jira; do not bundle assignee/status into the create call.
4. Use the new ticket as the association for those commits/PRs.

**Confirm with the user before creating tickets.** List proposed new tickets in a single batch and ask for approval (yes / edit / skip per item). If the user skips a proposal, mark that group's ticket as `(no-ticket: <branch-name>)` in the timeline — but **never** use a PR number as the ticket identifier.

#### Jira ticket creation flow (three calls, not one)

`createJiraIssue` is unreliable when you try to set assignee and status on creation. The fields silently get dropped, and a follow-up "modifying a ticket I just created" can also bump against permissions in some workspaces. Use this explicit sequence:

1. **Create** with `createJiraIssue`: project key, summary, description, issue type — **no assignee, no status**. Newly-created tickets land in the project's default status (typically "To Do").
2. **Assign** with `editJiraIssue`: set `assignee` to the user's Atlassian account ID. Doing this as a separate call is the only reliable way; don't trust assignee on creation.
3. **Transition** with `transitionJiraIssue` only if the work shipped today (commits merged, PR closed): move to "Done" or the project's equivalent terminal status. If the transition call is blocked by permissions, surface the failure to the user with the ticket key and the intended target status, and continue — do not retry silently.

For GitHub: `gh issue create --repo <slug> --title "..." --body "..." --assignee @me`, then close it with `gh issue close` only if the work merged.

**Tell the user upfront**, in the proposal batch, that newly-created Jira tickets will:
- land in the project's default status ("To Do" usually), and
- need a separate transition step to reach "Done", which may need extra permission.

This sets expectations before tickets are created, instead of surprising the user when transitions fail.

### 3a. The internal/ops pseudo-project

Not all real work belongs to a tracker project. Meetings without a project ticket, ticket triage, code review on others' PRs, process discussions, internal tooling/automation, 1:1s with no agenda — this is real time that the skill must account for, but creating a tracker ticket for each is wrong (clutters the project, often the user lacks a project for it).

If `internalProject` is configured (Section 0), use it as the destination for orphan time blocks that aren't code-with-tickets. Use the label (default `(internal)`) in the timeline's Ticket column. Examples:

| Signal | Ticket column |
|--------|---------------|
| Calendar event "Toggl automation discussion with Joshua" with no Jira ticket | `(internal)` (or `(calendar)` if you prefer to keep meetings separate) |
| 50-minute Slack thread reviewing someone else's PR in another team's repo | `(internal)` |
| Notion daily-log entry "process: ticket triage 14:00–14:45" | `(internal)` |
| Commits on a branch with no project association after the user declines a ticket proposal | `(no-ticket: <branch>)` (existing behaviour — code without tickets stays explicit) |

The distinction: `(no-ticket: <branch>)` is **code work that the user could have created a ticket for and chose not to**. `(internal)` is **non-code time that doesn't belong to any project**. Don't conflate them.

If `internalProject` is **not** configured but orphan non-code blocks exist, prompt the user once: "I have N minutes of non-project work today (meetings/discussions). Configure an internal pseudo-project to capture this, or label as `[unaccounted]`?"

### Hard rules

- **PRs are not tickets.** A PR is evidence *for* a ticket, never the unit of work itself. The `Ticket` column in the output must contain a tracker ticket key (e.g. `ACME-123`, `#45`), `(no-ticket: <branch-name>)` for skipped proposals, `(calendar)`, `(lunch)`, `(notion: <title>)`, `(carryover: <ticket>)`, or `[unaccounted]` — and nothing else. If you find yourself writing `PR #210` in that column, stop: you missed step 3.
- **One ticket per branch by default.** Don't split a branch's work into separate timeline rows just because it shipped as multiple PRs. Adjacent rows for the same ticket should be merged.
- **Don't invent ticket names.** Don't write things like `LCP investigation` or `Trim client bundle` as ticket identifiers — those are summaries. The ticket is the tracker key (or `(no-ticket: ...)` if the user skipped creation).

---

## 4. Build a Per-Ticket Timeline

For each ticket touched today, produce a `{start, end, ticket, summary, evidence[]}` block:

- **end** = timestamp of the last commit / PR / Notion edit activity for that ticket on this day. If the only signal is a Jira state change, use that timestamp.
- **start** = inferred earliest moment the user was working on this ticket today. Use the **earliest** of:
  - Earliest commit timestamp on that ticket today
  - Earliest file mtime among files changed in those commits (only if ≥ `dayStart`)
  - Earliest Slack message that day mentioning the ticket key, the branch, or related keywords
  - Earliest Notion page edit on that ticket (treat as end-of-work; subtract a 15min lead-in for the start signal)
  - Jira state change timestamp (e.g. moving to In Progress) — but only if it's *before* the earliest commit; state changes that happen *after* commits are not start signals.
  - If none of the above bound the start, default to **15 minutes before the first commit**.

- **evidence[]** = list of (kind, timestamp, ref) tuples that justify the block. This must appear in the final output so the user can audit.

Cap any single block at a sensible length — if `start` would be more than 3 hours before `end` with no intermediate evidence, shorten it to `end - 90min` and note the assumption.

### 4a. Carryover prompt (one-line, not a heuristic)

If there is a gap between `workdayDefaults.startTime` and the day's earliest evidence-backed block, **and** `previousDayLastTicket` is set, *do not* automatically insert a carryover block. Instead, surface a one-line prompt with the timeline:

> You last touched LW2-221 yesterday at 16:34. Continue from there to fill the 09:00–10:12 gap?

If the user confirms, insert a single block over the gap with `ticket = previousDayLastTicket.ticket` and the evidence entry `{kind: "carryover", timestamp: previousDayLastTicket.lastSignalAt}`. If they decline or ignore the prompt, leave the gap as `[unaccounted]`.

This replaces the previous multi-rule heuristic. A user knows immediately whether they continued from yesterday — asking is faster, more accurate, and less likely to mask a real signal we missed.

---

## 5. Combine & De-Overlap

Merge all per-ticket blocks plus calendar events into a single ordered timeline:

1. **Calendar events are fixed.** Truncate any work block that overlaps a calendar event to end at the event start, and resume after the event ends.
2. **Resolve work-block overlaps** by truncating the earlier block's `end` to the later block's `start` (the user can only do one thing at a time). When choosing which block "owns" the contested time, prefer the block with the most evidence in that window.
3. **Insert lunch** as a single fixed block of `lunchMinutes` near `lunchAroundTime`, snapping to a gap if one exists within ±60min of that time. If no gap exists, displace the lowest-evidence work block to make room.
4. **Bound the day** by `startTime` and `endTime`. Pull in the earliest block's start to no earlier than `startTime` and push the latest block's end to no later than `endTime` unless evidence (e.g. a commit at 19:00) clearly contradicts it — in that case keep the evidence and note the override.
5. **Fill gaps** > 30 minutes between work blocks with an `[unaccounted]` marker so the user can fill them in manually rather than silently extending neighbouring blocks.

---

## 6. Output

Render the timeline as a markdown table — **heading and table only, nothing else**. No Evidence section. No Caveats section. No narration of the construction process. See "Output discipline" at the top of this skill. The `Ticket` column must contain **only** one of: a tracker key (`ACME-123`, `#45`), `(no-ticket: <branch>)`, `(calendar)`, `(lunch)`, `(internal)`, `(notion: <title>)`, `(carryover: <ticket>)`, or `[unaccounted]`. PR numbers, branch names, and ad-hoc labels like "LCP investigation" are **not** valid values for this column — they belong in the Summary.

Evidence (timestamps, commit SHAs, PR URLs, mtime tuples) is gathered and used internally to anchor each row's `start`/`end`/`ticket`/`summary`. It is **not printed**. The user audits via the Toggl UI after Section 7, not via an evidence dump.

### Rounding and collapsing (avoid false precision)

Toggl entries shorter than ~15 minutes are noise. Showing `09m`/`12m` blocks creates an impression of timeline accuracy that the underlying signals don't support.

Apply these transforms to the rendered timeline (keep precise timestamps internally for Section 7):
1. **Round Start and End to the nearest 15 minutes** in the table — `15:47–15:56` becomes `15:45–16:00`. Compute `Duration` from the rounded values.
2. **Collapse adjacent same-ticket rows** separated only by sub-15-min cleanup blocks. Example: a `LW2-222` block 15:47–15:56 followed immediately by a `LW2-223` block 15:56–16:22 → emit as one row if the user confirms (or, if you can tell from internal evidence, fold the shorter one into the longer one with a merged summary).
3. **Drop sub-5-minute fragments** entirely after rounding — they round to a 0-minute row and add only noise.
4. After rounding, recompute durations and re-check that the day still adds up to within 30 min of `endTime - startTime - lunch`. If rounding pushes total drift past that threshold, ask the user about it before printing rather than printing a Caveats block.

Example (this is the **complete** output — heading, table, trailing prompt; nothing else):

```
# Toggl Tamer — 2026-05-06

| Start | End   | Duration | Ticket     | Summary                                              |
|-------|-------|----------|------------|------------------------------------------------------|
| 09:00 | 10:15 | 1h 15m   | ACME-204   | Implemented login form validation                    |
| 10:15 | 11:00 | 0h 45m   | (calendar) | Sprint planning                                      |
| 11:00 | 12:30 | 1h 30m   | ACME-211   | Fixed race condition in payment webhook              |
| 12:30 | 13:30 | 1h 00m   | (lunch)    | —                                                    |
| 13:30 | 15:00 | 1h 30m   | INT-17     | Drafted internal tools dashboard                     |
| 15:00 | 17:30 | 2h 30m   | ACME-204   | Reviewed feedback and merged login work              |

Apply edits, accept as-is, or regenerate?
```

### 6a. Pre-output self-check

Before printing, scan the rendered table and verify:

- [ ] Every `Ticket` column value matches one of the allowed forms above. If you see `PR #...`, a branch name, a phrase like "LCP investigation", or any free-text label, you have a bug — go back to Section 3 and create the missing ticket(s) (or mark them `(no-ticket: <branch>)` if the user declined).
- [ ] No two adjacent rows reference the same ticket without an intervening calendar/lunch/different-ticket row — merge them.
- [ ] Each non-meta row is internally backed by at least one signal (commit, PR, calendar, Notion, Slack, or carryover confirmation). Don't print this evidence — just verify it exists for every row.
- [ ] Start/End values are at 15-minute boundaries (Section 6 rounding applied). No `09m` / `12m` durations remain visible.
- [ ] Identity preflight (Section 0d) actually ran and `gitAuthorEmails` / `slackUserId` are present in config — not silently using `# userEmail` and `from:@me`.
- [ ] If any *signal source* returned zero hits today, **stop and ask the user before printing** rather than printing a Caveats footer. Common cause: an unconfigured author email or Slack ID.
- [ ] The output contains exactly: heading, table, trailing prompt. **No Evidence section, no Caveats section, no narrative paragraphs.** If any of those are present, delete them before printing.

If any check fails, fix the timeline and re-run the check. Do not output a timeline that fails this check.

After printing the timeline, **ask the user**: "Apply edits, accept as-is, or regenerate?" Loop on edits/regenerate until the user accepts. Once accepted, proceed to Section 7.

---

## 7. Write to Toggl

Run this section only when `togglEnabled: true` AND the user accepted the timeline in Section 6. If `togglEnabled: false`, print "Toggl write disabled — preview only." and stop.

### 7a. Check for existing entries on the target day

Before writing anything, call `mcp__toggl__get_time_entries` with `start_date` and `end_date` both equal to the target date. If the response is non-empty, **stop and prompt** the user with the existing entries listed (description, project, start, duration). Offer four options:

1. **Replace** — delete every existing entry for that day via `mcp__toggl__delete_time_entry`, then write the new timeline. Confirm the deletion list one more time before deleting.
2. **Keep and append** — leave existing entries alone, write the new ones alongside. Warn that this will likely produce overlapping entries; show the overlap count.
3. **Keep, skip overlapping** — for each new row whose `[start, end)` interval overlaps any existing entry, skip it. Write only the non-overlapping rows. Report which rows were skipped.
4. **Abort** — write nothing; exit cleanly.

If the response is empty, proceed directly to 7b without prompting.

### 7b. Map timeline rows to Toggl entries

Iterate the **rounded** timeline from Section 6. For each row:

- Skip rows whose `Ticket` value is in `togglWrite.skipMeta` (default: `(lunch)`, `[unaccounted]`).
- Resolve the Toggl project name:
  - For rows tied to a configured project → that project's `togglProjectName`.
  - For `(internal)` rows → `internalProject.togglProjectName`.
  - For `(calendar)`, `(notion: ...)`, `(no-ticket: ...)`, `(carryover: ...)` → use the project of the most relevant signal in the row's evidence; if none can be determined, fall back to `internalProject.togglProjectName`. If that's also unset, prompt the user once for a Toggl project name to use as the catch-all and cache it as `togglWrite.fallbackProjectName` in config.
- Build the `description`: `<TicketKey> — <Summary>` for ticket rows; `<Summary>` alone for `(calendar)`/`(internal)`/`(notion: ...)` rows. Trim to ≤ 200 chars.
- Compute `start` as an ISO 8601 timestamp **with the configured timezone offset** (not UTC, not naive). Example: `2026-05-06T09:00:00+10:00` for Brisbane.
- Compute `duration_minutes` from `End - Start` of the rounded row. If the result is < 1 minute, drop the row and note it.
- If `togglWrite.tagWithTicket` is true and the ticket value is a real tracker key (matches `^[A-Z]+-\d+$` or `^#\d+$`), pass it as a tag.

### 7c. Confirm the write batch

Print the proposed Toggl write as a compact table — one row per planned entry — and ask: "Write these N entries to Toggl? (yes / edit / abort)". Show the total minutes about to be written and the count, so the user can sanity-check against the day's accounted total. Do not proceed without explicit `yes`.

### 7d. Execute the writes

Call `mcp__toggl__create_time_entry` once per row, sequentially (not in parallel — Toggl rate-limits and an out-of-order failure is hard to reason about mid-batch). For each call, capture the returned entry ID. If a call fails:

1. Stop the batch immediately. Do not continue writing further entries.
2. Print which rows succeeded (with their entry IDs) and which row failed (with the error).
3. Ask the user whether to **roll back** (delete the successful entries via `mcp__toggl__delete_time_entry`), **leave as-is** (partial write), or **retry from the failed row**.

### 7e. Confirm and link

After a successful batch, print a summary: count of entries written, total minutes, and a link to the user's Toggl day view (`https://track.toggl.com/timer` is fine — Toggl doesn't have a stable per-day URL). Suggest the user double-check the Toggl UI before closing the loop.

### Hard rules for Section 7

- **Never write without an accepted timeline.** If the user is still iterating in Section 6, do not proceed.
- **Never silently overwrite existing entries.** If 7a finds entries on the target day and the user doesn't pick "Replace", existing entries are off-limits.
- **Always include a timezone in `start`.** Naive timestamps are rejected by the MCP. Use `workdayDefaults.timezone` to construct the offset.
- **Sequential writes only.** Parallel `create_time_entry` calls are not safe for partial-failure reasoning.
- **Surface the project-name mismatch loudly.** If a row's resolved Toggl project doesn't exist in Toggl at write time (it was deleted/renamed since preflight), abort the batch — don't write the entry projectless.

---

## Quick Reference

| Phase | Inputs | Outputs |
|-------|--------|---------|
| 0. Config + auto-discovery | `config.json` or probes | Project list, defaults |
| 0d. Identity preflight | Repos, MCPs | Cached `gitAuthorEmails`, `slackUserId`, etc. |
| 1. Window | Date arg | `dayStart`, `dayEnd` ISO |
| 2.0. Notion daily-log | `notion-search` + `notion-fetch` | Ground-truth narrative for the day |
| 2. Signals | MCP + git + gh | Raw events |
| 3. Associate (incl. 3a internal) | Events + tickets | `{event → ticket}` map |
| 4. Per-ticket (incl. 4a carryover prompt) | Mapped events | Blocks with evidence |
| 5. Merge | Blocks + calendar | Non-overlapping timeline |
| 6. Output (rounded to 15min) | Timeline | Markdown table + evidence |
| 7. Write to Toggl | Accepted timeline + Toggl MCP | Created Toggl entries (after existing-entry resolution) |

## Common Pitfalls

- **Do not assume commit time = work time.** Commits are end-of-work signals. Use file mtimes / Slack / state changes to find the start.
- **Jira state changes after commits are not start signals.** Many users move tickets to "In Progress" only when they go to commit. Only use state changes as start signals when they precede the earliest commit.
- **Timezones** — all signal sources return UTC by default; normalise to the configured timezone before comparing or rendering.
- **Don't silently create tickets.** Always confirm with the user before calling `createJiraIssue` or `gh issue create`.
- **Notion edits are end-of-work signals**, just like commits. Don't anchor block `start` to the edit timestamp; subtract a lead-in or use other evidence.
- **Filter Notion edits by the user's Notion identity.** Page `last_edited_by` is a Notion user ID, not an email — resolve once via `notion-get-users` and cache as `notionUserId` in config.
- **Continuation blocks are assumptions, not evidence.** Always label them as `kind: "carryover"` in the evidence list so the user can see the difference between observed work and inferred handoff. Never use a carryover block to extend a real block's duration.
- **Don't extend blocks across calendar events.** Truncate at the event boundary; resume after.
- **Don't fill every gap.** If you have no evidence, mark `[unaccounted]` and let the user decide.
- **Single-author filter** — when multiple people use the same machine or repo, always filter by every email in `userIdentities.gitAuthorEmails`, not just `# userEmail` and never `--author=$(git config user.name)`. The user's work email is often *not* the git author email — using the wrong one returns zero rows silently.
- **Never use `from:@me` in Slack searches** — it silently returns nothing in many workspaces. Use `from:<@SLACK_USER_ID>` with the cached `userIdentities.slackUserId`. Self-DMs are a key signal — search them explicitly with `from:<@U…> to:<@U…>`.
- **Read the Notion daily-log first.** A user-curated daily log is more accurate than any reconstruction. Check Section 2.0 *before* gathering other signals — don't spend 30 tool calls rebuilding a timeline the user already wrote down.
- **Discover Jira projects from commit messages, not just JQL.** Users who don't move tickets in Jira are invisible to assignee/reporter/comment queries. Grep recent `git log` for `[A-Z]+-\d+` patterns to surface project keys regardless of Jira activity.
- **Don't trust file mtimes blindly** (Section 2g). Spotlight, format-on-open, and `git checkout` all rewrite mtimes wholesale. If all candidate mtimes cluster within ±5min on a non-target day, drop them.
- **Jira ticket creation is three calls, not one.** `createJiraIssue` silently drops assignee/status. Always follow with `editJiraIssue` to set assignee, then `transitionJiraIssue` only if the work shipped today. Tell the user upfront that transitions may need extra permission.
- **Internal/non-project time has a home.** Use the `(internal)` pseudo-project for meetings, ticket triage, code review on others' PRs, and process discussions. Without it, real time is dropped from the timeline.

## Red Flags — Stop and Ask

- Config file missing → run interactive setup, do not guess
- No commits, no PRs, no calendar events → ask the user if it was a working day before fabricating a timeline
- A proposed new ticket would land in a project the user did not configure → skip, surface to user
- Total accounted time differs from `endTime - startTime - lunch` by more than 90 minutes → flag it; do not stretch evidence to fill the gap
- About to write `PR #N`, a branch name, or a free-text phrase in the `Ticket` column → stop. The output is ticket-centric. Either find the existing ticket, propose a new one for user approval, or use `(no-ticket: <branch>)` if the user declined. PR numbers belong in evidence, not in the ticket column.
- About to proceed with a missing integration (Calendar, Slack, Notion, Jira, GitHub) without explicit user approval → stop and run the Section 0e MCP preflight prompt. Missing integrations are surfaced *before* signal gathering, not as a footnote after the timeline.
- About to filter `git log` by `# userEmail` alone → stop. Use `userIdentities.gitAuthorEmails` (resolve via Section 0d if not cached). Wrong email = silent zero results = confidently-wrong "you did nothing" output.
- About to use `from:@me` in any Slack search → stop. Use `from:<@SLACK_USER_ID>`. The `@me` modifier silently returns nothing in many workspaces.
- Skipping Section 2.0 (Notion daily-log) when `notionDailyLog.enabled` is true → stop and read the daily log first. Reconstructing from 30 signals what the user already wrote down by hand wastes tool calls and produces a less accurate timeline.
- About to call `mcp__toggl__create_time_entry` while existing entries already exist for that day → stop. Run Section 7a (Replace / Keep+append / Keep+skip-overlap / Abort) first.
- About to call `mcp__toggl__create_time_entry` with a `start` value that has no timezone offset, or with `duration_minutes <= 0` → stop. The MCP will reject it; build the timestamp from `workdayDefaults.timezone` and the rounded `Start`/`End` from Section 6.
- About to call `start_timer` to back-fill a past day → stop. `start_timer` only starts a *running* timer at the current moment. Use `create_time_entry` with explicit `start` and `duration_minutes` instead.
- About to print an interim/draft timeline, a "let me reconsider" passage, UTC↔local conversion math, "Section 6 rounding & collapsing" headers, "Self-check" output, an Evidence section, a Caveats section, or any narration of the construction process → stop. See "Output discipline" at the top of this skill. The user wants exactly: heading, table, trailing prompt — nothing else.
