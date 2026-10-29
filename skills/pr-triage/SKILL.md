---
name: pr-triage
description: "Batch-review open PRs in parallel, triage into merge/fix/follow-up buckets, then execute each bucket with per-action human gates on every irreversible operation. Parallel reviews in worktrees compress wall-clock; sequential human-gated merges and diff-only fixes keep blast radius bounded."
argument-hint: "[--prs <N,N,...>] [--limit <N>] [--repo <owner/repo>] [--auto-merge] [--skip-fix] [--prefilter | --no-prefilter] [--sort oldest|newest] [--release]"
category: "Build & ship"
when-to-use: "When 2+ PRs are open and you want to review, triage, merge, and fix them in a single session. Triggers on 'triage PRs', 'review open PRs', 'batch review', 'clear the PR queue'."
skills: [review, fix-pr, contract]
flags: [--auto-merge, --skip-fix, --prs, --limit, --repo, --prefilter, --no-prefilter, --sort, --release]
max_tool_use_iterations: 100
failure_modes:
  - RATE_LIMIT_429: "Dispatching 5+ review agents in one wave triggers Anthropic 429 rate limits. Mitigation: cap at 5 concurrent; larger batches run in waves of 5."
  - JEV_PREFILTER_UNAVAILABLE: "Jev MCP server unreachable, API key missing, or server refuses diff paths. Mitigation: fail open; every PR goes to the Full review route."
  - MERGE_CONFLICT_AFTER_SIBLING: "Merging one GREEN PR makes a later sibling conflict with the updated base. Mitigation: conflict-minimizing merge order via merge-tree simulation; conflicting PRs move to BLOCKED for a fix-pr dispatch."
  - FIX_AGENT_STALE_COMMENT: "/fix-pr harvests PR comments, but a stale comment from an earlier round may not reflect current state. Mitigation: pr-triage posts blocking findings with the reviewed SHA before dispatching /fix-pr; /fix-pr probes re-verify each finding on HEAD."
  - REVIEW_FINDINGS_INVISIBLE_TO_FIX: "/review is read-only and never posts, so its findings exist only in the coordinator's context, while /fix-pr builds its fix spec from GitHub stores. Mitigation: Wave 2B posts the blocking set as a PR comment (gh pr comment --body-file) before dispatching /fix-pr, which /fix-pr's Phase 1 source 3 (issue-level comments) already harvests."
  - GH_AUTH_MISSING: "gh CLI is not authenticated. Mitigation: fail closed in Wave 0 before any work begins."
  - SCRATCH_DIR_COLLISION: "Two concurrent pr-triage sessions share scratch paths and corrupt each other's state. Mitigation: per-run scratch directory with timestamp and random suffix."
  - UNREVIEWED_HEAD_MERGE: "PR head moves after review but before merge, landing unreviewed commits. Mitigation: --match-head-commit on every merge; GitHub refuses if the head moved."
  - DUPLICATE_ADVISORY_ISSUES: "Wave 2C creates advisory issues that duplicate existing open issues. Mitigation: session-unique tag in title; exact title-prefix match search; recent-issue search (last hour) before creating; comment on existing issue instead, or skip creation."
  - CONCURRENT_RUN_MERGE_RACE: "Two parallel pr-triage runs on the same repo can both pass the pre-merge state check before either completes gh pr merge. Mitigation: Wave 0 warns when a sibling pr-triage session is detected via list_sessions; --match-head-commit is the last-resort hard guard (GitHub rejects the second merge if the head moved). The TOCTOU window between the final state check and the merge call is accepted as irreducible without repo-side locking."
  - CONCURRENT_RUN_ISSUE_DEDUP_RACE: "Two parallel runs can both search for an existing issue, find nothing, and both create the same follow-up issue. Mitigation: session-unique tag in the issue title makes each run's issues distinct; recent-issue search (last hour) before filing catches a sibling's just-created issue."
---

# PR Triage — decision-support orchestrator

You are executing a PR triage session. This skill reviews open PRs in parallel,
triages results into actionable buckets, and executes each bucket with human gates
on every irreversible operation.

## Hard constraints

1. **No irreversible external write without human confirmation.** Merges, pushes,
   issue creation, and GH comment posting each require explicit operator approval.
   The only exception: `--auto-merge` flag opts into batch-merge of green PRs
   (still sequential with CI check between each).
2. **Reviews are read-only.** `/review` never posts to GitHub. This skill owns
   the decision of what to post and when.
3. **Fix agents push directly.** Dispatch `/fix-pr` (no `--no-push`) so fixes
   are pushed to the PR branch immediately after passing the test gate.
4. **Sequential merges only.** Never merge two PRs in parallel. After each merge,
   verify CI / build before proceeding to the next.
5. **Rate-limit awareness.** Cap concurrent review dispatches at 5. For >5 PRs,
   run in waves of 5.

## Arguments

Parse these from `$ARGUMENTS`:

| Flag | Default | Meaning |
|------|---------|---------|
| `--prs 101,102,105` | (all open) | Specific PR numbers to triage |
| `--limit N` | 10 | Max PRs to fetch when using all-open discovery |
| `--repo owner/repo` | (cwd git remote) | Target repository |
| `--auto-merge` | off | Skip human gate for merge of green PRs (still sequential + CI-gated) |
| `--skip-fix` | off | Skip Wave 3 fix phase; just triage and merge |
| `--prefilter` | on for public repos | Force the Wave 0.5 Jev pre-filter on, including for private repos |
| `--no-prefilter` | off | Skip Wave 0.5; every PR gets a full review |
| `--sort oldest\|newest` | newest | PR selection order when using all-open discovery. `newest` (default) triages recently opened PRs first; `oldest` clears the backlog |
| `--release` | off | After all merges complete, draft release notes from PRs merged this session and offer to create a GitHub release. The operator provides the tag at the approval gate; `--auto-merge` does not bypass this gate. |

## Execution

### Wave 0 — Preflight (inline, no dispatch)

1. Verify `gh` CLI is authenticated: `gh auth status`. **Fail closed** if not.
   Verify `gh` version is 2.41 or later (`gh --version`); versions before 2.41
   lack `--match-head-commit` and will fail at merge time. **Fail closed** if not met.
   Verify `git` version is 2.38 or later (`git --version`); versions before 2.38
   lack `merge-tree --write-tree` used in Wave 2A merge-order simulation.
   **Fail closed** if not met.
2. Determine target repo from `--repo` flag or `gh repo view --json nameWithOwner`.
2a. **Concurrent-run detection (warn-only).** Call `list_sessions` and scan the
    results for any other live session whose prompt or skill invocation references
    the skill name `pr-triage`. A session matches if its text contains the literal
    string `pr-triage`; repo matching is not required (repo context is often
    absent mid-session). **Note:** this detection window is narrow — sessions may
    start or finish between the `list_sessions` call and the merge attempt, so
    false negatives are possible. `--match-head-commit` (Wave 2A step 3) is the
    hard backstop for any miss. A false positive is acceptable
    (the warning is advisory). If any such session is found, print a prominent
    warning:

    ```
    ⚠️  WARNING: another pr-triage session appears to be running (session <id>);
    it may be targeting this repo (<repo>). Concurrent runs can race on merges
    and issue filing. Recommended: cancel one session before proceeding.
    Continuing in 10 seconds — press Ctrl-C to abort.
    ```

    Then call the `bash` tool with `sleep 10` to pause before continuing.
    This is advisory — do not hard-gate on it.
    False positives are possible (e.g. a session that already finished but
    appears live); the warning is advisory, not a hard gate. If `list_sessions`
    is not available in the current tool surface, skip this step silently.
3. Create the per-run scratch directory used by all later waves. Generate the name
   **once** with:
   `echo ".afk/tmp/pr-triage-$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 3)"`
   Record the literal string it prints (e.g. `.afk/tmp/pr-triage-20240101-120000-a1b2c3`);
   this is the **run dir**. Use that literal string — never a shell variable — in
   every subsequent command, tool argument, and file path that references it.
   Run `mkdir -p <run-dir>` immediately.
4. Fetch open PRs:
   - If `--prs` provided: `gh pr view <N> --json number,title,headRefName,headRefOid,url,author,reviewDecision,statusCheckRollup,isDraft` for each.
   - Otherwise: `gh pr list --state open --limit <limit> --json number,title,headRefName,headRefOid,url,author,reviewDecision,statusCheckRollup,isDraft,createdAt`.
     When `--sort oldest`, sort the fetched list by `createdAt` ascending before proceeding. Default order (newest first) needs no re-sort.
   Record each PR's `headRefOid` as `fetched_sha` for later staleness checks.
5. If 0 PRs found, report Done with "No open PRs to triage" and stop.
6. Print the PR manifest table to the operator:

```
## PR Triage — <repo> — <N> open PRs

| # | Title | Author | Branch | CI | Review |
|---|-------|--------|--------|----|--------|
| 101 | Fix auth timeout | dependabot | fix/auth | ✅ | APPROVED |
| 102 | Add dark mode | griffin | feat/dark | ❌ | CHANGES_REQUESTED |
| ... | ... | ... | ... | ... | ... |

Proceeding to parallel review wave...
```

### Wave 0.5 — Jev pre-filter (inline, optional, fail-open)

Decide how much review each PR gets before paying for it. Jev (TypeSafe AI's
judgment engine, exposed by the `jev` MCP server) reads each PR's diff
server-side and answers three narrow questions with calibrated probabilities.
The diffs never enter this session's context.

**Jev only decides whether a PR is reviewed now, how deeply, and in what order.**
It never marks a PR GREEN or BLOCKED: those still come only from a `/review`
verdict, and every merge keeps its gate. Jev judges what a diff looks like, not
whether it is correct.

**Run only when all of these hold**, otherwise print
`Pre-filter: off (<reason>)` and go straight to Wave 1 with every PR on the
Full route:

- `mcp__jev__jev_triage` is in your tool list.
- `--no-prefilter` is not set.
- The repo is public (`gh repo view <repo> --json visibility`), or
  `--prefilter` is set. Private repos are off by default because the pre-filter
  sends their diffs to TypeSafe's API.

**Steps:**

1. For each PR write its title, body, changed-file list, and diff to
   `<run-dir>/<N>.txt` under the current working directory, using shell
   redirection only (for example
   `{ gh pr view <N> --repo <repo> --json title,body,files --jq '.title, .body, (.files[].path)'; gh pr diff <N> --repo <repo>; } > <run-dir>/<N>.txt`).
   Do not read these files yourself. They must sit under the working directory,
   because the Jev server only reads paths below the directory it started in.
2. Make **one** `mcp__jev__jev_triage` call with one item per PR
   (`{ "id": "<N>", "path": "<run-dir>/<N>.txt" }`),  
   `yes_at_or_above: 0.85`, `no_at_or_below: 0.2`, and these questions:
   - `trivial` (check): "Is every change in this pull request limited to
     documentation, comments, formatting, lockfiles, or dependency version
     numbers, with no change to executable logic, configuration behavior, or
     tests?"
   - `ready` (check): "Does this pull request look like finished work that is
     ready for review, with no WIP or draft markers, placeholder code, or TODOs
     describing unfinished parts of the change?"
   - `risk` (score), levels lowest first:
     0. "Touches no security-sensitive or widely shared code."
     1. "Touches internal code shared by several modules."
     2. "Touches a public API, persistence or migrations, or concurrency."
     3. "Touches authentication, credentials, permissions, secret handling, or payments."
3. Route each PR:

   | Route | Rule | Wave 1 treatment |
   |-------|------|------------------|
   | **Light** | `trivial` verdict is yes AND `risk` score < 1.0 | `skill review <pr-url> --light`, model `sonnet` |
   | **Deferred** | (`isDraft` is true OR `ready` verdict is no) AND the PR was not named in `--prs` | Not reviewed; lands in SKIP with reason `deferred` |
   | **Full** | Everything else, including every PR whose item errored | Unchanged: `skill review <pr-url>`, model `claude-opus-4-6` |

   If a PR matches both Light and Deferred, Deferred wins.
4. **Fail open.** If the call errors, the server refuses the paths, or an item
   returns an `error` (for example a diff over Jev's size limit), that PR goes
   Full. Never retry by passing the diff as `text`: that pulls it into context,
   which is what this wave exists to avoid.
5. Print the routing table under the manifest, then continue to Wave 1 without
   waiting. Routing is read-only, and the operator can pull any deferred PR back
   in at Wave 1.5.

```
## Pre-filter (Jev)

| # | Route | P(trivial) | P(ready) | Risk (0-3) |
|---|-------|------------|----------|------------|
| 101 | Light | 0.93 | 0.97 | 0.2 |
| 102 | Full | 0.04 | 0.91 | 2.6 |
| 108 | Deferred | 0.01 | 0.12 | 1.1 |
```

### Wave 1 — Parallel Review (subagent fan-out)

Dispatch one `agent` call per PR on the Full or Light route, in parallel.
Deferred PRs are not dispatched. When Wave 0.5 ran, dispatch in descending
`risk` order so the riskiest PRs are reviewed first if the session is cut short.
A PR with no `risk` score (its item errored) counts as highest risk.

- Full route: `skill review <pr-url>`, model `claude-opus-4-6`
- Light route: `skill review <pr-url> --light`, model `sonnet`
- Cap at 5 concurrent agents. If >5 PRs, run sequential waves of 5 (dispatch the first batch, await all results, then dispatch the next batch).
- These are read-only reviews — no `isolation: "worktree"` needed.

**For each review result, extract:**
- The `Decision: MERGE / DO NOT MERGE` verdict line
- The blocking findings count and severity breakdown
- The full findings list (including non-blocking advisory/low/nit findings)

### Wave 1.5 — Triage Gate (inline, no dispatch)

**Re-fetch PR state before classifying.** For every PR that was reviewed (Full or
Light route), run
`gh pr view <N> --json state,headRefOid,mergedAt,closedAt`.
- `state` is `MERGED` or `CLOSED` → put the PR into **SKIP** with reason
  (e.g. "merged elsewhere by @actor at <mergedAt>"). Do not classify it further.
- `headRefOid` differs from the `fetched_sha` recorded in Wave 0 → the head
  moved after the review started. Mark the review **stale** and put the PR into
  **SKIP** with reason "head moved (<old_sha7> → <new_sha7>); re-triage to
  review the new head".

Classify each remaining PR into exactly one bucket based on the review verdict:

| Bucket | Criteria |
|--------|----------|
| **GREEN** | `Decision: MERGE` — zero blocking findings |
| **BLOCKED** | `Decision: DO NOT MERGE` — has blocking findings. Dispatch `/fix-pr` to address them. |
| **SKIP** | PR already merged/closed during review, review failed/timed out, or deferred by the Wave 0.5 pre-filter (not reviewed) |

Present the triage table to the operator:

```
## Triage Results

| # | Title | Bucket | Blocking | Summary |
|---|-------|--------|----------|---------|
| 101 | Fix auth timeout | 🟢 GREEN | 0 | Clean — 2 low, 1 nit |
| 102 | Add dark mode | 🔴 BLOCKED | 2 high | Missing error handling in theme.ts:41, untested edge case |
| 103 | Bump deps | 🟢 GREEN | 0 | Dep-bump, no issues |
| 104 | Refactor auth | 🔴 BLOCKED | 1 critical | API contract change needs fix |
| 108 | WIP: new exporter | ⚪ SKIP | — | Deferred by pre-filter (P(ready) 0.12), not reviewed |

### Proposed actions:
- **Merge:** #101, #103 (sequential, CI-gated)
- **Comment:** Post blocking findings to each BLOCKED PR before fix dispatch
- **Fix:** #102, #104 (dispatch /fix-pr, push after test gate)
- **Issue (advisory):** #101 has 2 low findings, #103 has 1 nit — track after merge

Awaiting your approval to proceed. Reply with:
- "go" or "yes" — execute all proposed actions
- "merge only" — only merge green PRs
- "fix only" — only fix blocked PRs
- "skip <number>" — exclude a specific PR
- "include <number>" — full-review a deferred PR now, then re-present this table
- Any custom instruction
```

**Wait for operator approval** via `ask_question` before proceeding to Wave 2.
If on a non-interactive surface (daemon, one-shot), **stop here** — emit the
triage table as the terminal artifact and report Done.

**Re-fetch after approval.** The operator may take minutes or hours to respond.
After approval returns, re-fetch every PR still in GREEN or BLOCKED:
`gh pr view <N> --json state,headRefOid`.
- `state` is `MERGED` or `CLOSED` → move to SKIP; report
  "#<N> was <merged/closed> while waiting for approval".
- `headRefOid` differs from `fetched_sha` → move to SKIP with reason
  "head moved after review; re-triage needed".
If any PR moved, re-print the updated triage table before proceeding.

### Wave 2 — Execute Buckets (sequential, human-gated)

Execute buckets in this order: Merge → Fix. Each bucket is a sequential
sub-wave. Never overlap buckets.

#### 2A — Merge Green PRs

**0. Plan a conflict-minimizing merge order (local, no writes to any remote).**
Merging green PRs in arbitrary order lets an early merge knock a later,
otherwise-clean PR into conflict, which then costs a rebase, a gate re-run, and
often a second rebase when the next sibling lands. Simulate the batch first:

```bash
git fetch origin <base> && for n in <green PRs>; do git fetch origin pull/$n/head:refs/pr-sim/$n; done
sim=$(git rev-parse origin/<base>)
# For each PR, smallest diff first (gh pr view <N> --json additions,deletions):
if tree=$(git merge-tree --write-tree "$sim" refs/pr-sim/<N>); then
  sim=$(git commit-tree "$tree" -p "$sim" -p refs/pr-sim/<N> -m sim)   # clean → keep in order
else
  : # conflicts with the batch so far → move <N> to the END of the order
fi
```

Merge in the resulting order: clean PRs first, conflicting ones last. A PR that
conflicts in simulation is still attempted last (its conflict may be with a PR
the operator later declines), and goes to BLOCKED by step 1 if it really does
conflict. Show the planned order and the simulated conflicts in the merge
confirm. Requires git ≥ 2.38 (`merge-tree --write-tree`), enforced by Wave 0
preflight. Delete the `refs/pr-sim/*` refs afterwards.

For each GREEN PR in that order, **sequentially** (never parallel):

1. Re-check state and mergeability:
   `gh pr view <N> --json state,headRefOid,mergeable,mergeStateStatus`.
   - `state` is not `OPEN` → SKIP ("merged/closed by another actor"); **continue**
     to the next PR instead of stopping the merge loop.
   - `headRefOid` differs from `fetched_sha` → do not merge automatically.
     Inform the operator: "#<N> head moved (<old_sha7> → <new_sha7>) after
     review; the new commits are unreviewed. To proceed safely, re-triage."
     **Gate:** if the operator explicitly names the new SHA and approves merging
     it unreviewed, re-fetch `headRefOid` to confirm the PR has not moved
     again, then merge with `--match-head-commit <new_sha>` and note
     "merged unreviewed SHA <new_sha7>" in the terminal summary. Otherwise
     move to SKIP.
   - Not mergeable (conflict from sibling merge) → move to BLOCKED bucket
     and inform operator.
2. Unless `--auto-merge` is set, confirm with operator:
   `"Merge #<N> '<title>' into <base>? (y/n)"`
   `--auto-merge` does not cover PRs on the Light route: always confirm those
   individually, since they were reviewed on a cheaper model.
3. Merge: `gh pr merge <N> --merge --delete-branch --match-head-commit <fetched_sha>`
   (Use `--merge` not `--squash` unless the PR is single-commit; respect repo's
   merge strategy if detectable from `gh api repos/{owner}/{repo}` settings.)
   **`--match-head-commit` is the last-resort guard against merge races.** If a
   concurrent pr-triage run (or any other actor) merges this PR or pushes new
   commits between step 1's state check and this merge call, GitHub will reject
   the merge because the head SHA no longer matches `<fetched_sha>`. This
   eliminates the risk of landing an unreviewed commit. The TOCTOU window between
   step 1 and this call is accepted as irreducible without repo-side locking;
   `--match-head-commit` is the hard backstop that makes it safe.
4. **After each merge**, verify the base branch CI:
   - `gh pr checks` on the next candidate (if any) — if the base branch now
     shows failures, **stop merging** and inform the operator.
   - Brief pause (5s) to let CI pick up the new base.

#### 2B — Fix Blocked PRs

**Before dispatching `/fix-pr`, post the blocking findings as a PR comment.**
`/review` is read-only and never posts to GitHub, so the blocking findings exist
only in the coordinator's context. `/fix-pr` builds its fix spec from GitHub
stores (inline review comments, issue-level comments, CI checks); without this
post, fix agents fly blind on the actual blockers. This is the same pattern
`/land` uses in Phase 1e.

For each BLOCKED PR:

1. Write the blocking findings from the Wave 1 review result to `<run-dir>/findings-<N>.md`. Use `write_file`
   or a quoted heredoc — never inline `--body` or double-quoted shell strings.
   The heading must be:
   `pr-triage: <n> blocking findings @ <reviewed_sha7>`
   followed by one numbered item per finding: severity, `file:line`, finding
   sentence, suggestion. (`<reviewed_sha7>` is the first 7 chars of the
   `headRefOid` captured in Wave 1.)
2. Post the comment:
   `gh pr comment <N> --body-file <run-dir>/findings-<N>.md`
3. Then dispatch `/fix-pr`:

**Note:** For 3+ blocked PRs, posting comments sequentially before any fix dispatch adds N × round-trip latency. At typical scale (≤5 PRs) this is acceptable; for larger batches, post all comments first as a group before dispatching any fix agents. **For batches above 5, complete step 2 for all PRs before beginning any step 3 dispatches** — this overrides the per-PR loop ordering above *(latency reduction — independent of the Wave 1 rate-limit cap of 5)*.

- If only 1 blocked PR: dispatch a single `agent` call: `skill fix-pr <N>`
- If 2+ blocked PRs: dispatch parallel `agent` calls, each calling `skill fix-pr <N>`.
  `/fix-pr` handles its own worktree creation internally, so no `isolation: "worktree"` is needed on the dispatch.
- Max 3 concurrent fix agents (narrower than review wave — fixes are heavier).
- PRs moved here only because a sibling merge made them conflict: dispatch them
  after 2A has finished, once, so each is rebased onto the final base rather than
  once per sibling merge. Tell the fix agent: resolve code conflicts first, then,
  if the repo documents a regenerate command for generated or baseline files (check
  AFK.md / CLAUDE.md / CONTRIBUTING / `.gitattributes`; agent-afk has
  `pnpm regen:generated`), run it instead of hand-merging those files.

`/fix-pr` handles its own worktree creation, test gate, push, and cleanup.
After all fix agents return, present results:

```
## Fix Results

| # | Title | Status | Summary |
|---|-------|--------|---------|
| 102 | Add dark mode | ✅ Fixed + pushed | Added error handling + test |
| 107 | Migrate config | ❌ Blocked | Needs upstream API change (worktree kept) |
```

#### 2C — Track Advisory Findings from Merged GREEN PRs

After merging GREEN PRs (Wave 2A), create tracking issues for any non-blocking
findings (low, advisory, nit) that the review surfaced. These findings were not
severe enough to block merge but represent real improvement opportunities that
will otherwise be lost in the triage session transcript.

**Skip when:** a GREEN PR's review had zero non-blocking findings (truly clean).

For each GREEN PR with non-blocking findings, draft a GitHub issue:
- **Title:** `review: advisory findings from #<N> — <PR title summary> [triage:<run-id>]`
  where `<run-id>` is the unique suffix of the run dir recorded in Wave 0 step 3
  (e.g. `20240101-120000-a1b2c3`). Within a single run, sequential filing order
  is the dedup mechanism — each issue is filed one at a time, so a duplicate can
  be detected before the next is created. The `[triage:<run-id>]` tag makes each
  run's issue titles distinct; the recent-issue search below catches
  concurrent-run duplicates regardless of the tag.
- **Body:** the non-blocking findings list (severity, file/line, description,
  suggestion), the merged PR link, and a note that these were reviewed and
  accepted at merge time.
- **Labels:** if the repo has a `review-followup` or `tech-debt` label, apply it.

Batch-present all drafted issues to the operator for approval:

```
## Advisory Follow-ups from Merged PRs

### Issue for PR #101 — Fix auth timeout
**Title:** review: advisory findings from #101 — fix auth timeout [triage:20240101-120000-a1b2c3]
**Findings:** 2 low (missing null check in fallback path, stale comment)

### Issue for PR #103 — Bump deps
**Title:** review: advisory findings from #103 — bump deps [triage:20240101-120000-a1b2c3]
**Findings:** 1 nit (changelog entry formatting)

Create these issues? (y/n/edit/skip)
```

On approval, **deduplicate before creating.** For each drafted issue, run two
searches and skip creation if either finds a match:

1. **Exact-title search** — open issues with this PR's base prefix:
   `gh issue list --state open --search "\"review: advisory findings from #<N>\" in:title" --limit 5 --json number,title,createdAt`.
   Confirm an exact title-prefix match: the returned issue title must begin with
   `review: advisory findings from #<N>`. A loose keyword match is not sufficient.

2. **Recent-issue search** — issues created in the last hour (catches a concurrent
   run that filed the same issue moments ago). First, compute the cutoff time in
   a standalone bash call:
   `date -u -v-1H +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u --date='1 hour ago' +%Y-%m-%dT%H:%M:%SZ`
   If the command outputs an empty string (e.g. on Alpine/busybox where neither
   BSD nor GNU date flags are available), skip search 2 entirely rather than
   issuing a malformed query. Otherwise, substitute the result into the search:
   `gh issue list --state open --search "\"review: advisory findings from #<N>\" in:title created:><CUTOFF>" --limit 5 --json number,title,createdAt`
   where `<CUTOFF>` is the literal datetime string from the previous command.
   If any issue is returned, treat it as a duplicate regardless of whether the
   `[triage:…]` tag matches — a concurrent run used a different run-id.

Write every body file with the `write_file` tool, or with a quoted heredoc
redirected to the file (`cat > <run-dir>/issue-<N>.txt <<'EOF'`). Never place a
markdown body inside double quotes in a shell command (for example
`printf '%s' "<body>"` or `echo "<body>"`): the shell expands `$` and backticks
in the body and silently corrupts the code spans.
If a confirmed match exists, write the non-duplicate findings to
`<run-dir>/comment-<N>.txt`, then post:
`gh issue comment <existing> --body-file <run-dir>/comment-<N>.txt`
and note `existing #<existing>` in the terminal summary. Otherwise write the
draft body to `<run-dir>/issue-<N>.txt`, then create:
`gh issue create --title "..." --body-file <run-dir>/issue-<N>.txt`.
Write all issue body drafts and comment body files to `<run-dir>/` so they
are cleaned up with the rest of the run's scratch files.
On "skip": omit without creating. Advisory issues are optional — the operator
may reasonably skip all of them.

### Wave 3 — Re-review Fixed PRs (conditional)

If any fixes were pushed in Wave 2B AND `--skip-fix` is not set:

1. For each pushed fix, dispatch `/review <pr-url>` (single agent, not parallel —
   there are typically 1-2 fixed PRs).
2. If the re-review returns GREEN, inform operator and offer to merge.
3. If still BLOCKED, report findings and stop — do not loop.

### Wave 4 — Release (conditional, `--release` only)

If `--release` is set and at least one PR was merged in this session (Wave 2A or
Wave 3; `--skip-fix` suppresses Wave 3 but not Wave 4, so Wave 2A merges alone
satisfy this condition):

1. Collect all PRs merged this session: number, title, and author.
2. Fetch the latest existing version tag from GitHub using the git refs API
   (catches bare tags pushed by CI as well as tags backed by a GitHub Release):
   ```bash
   gh api repos/{owner}/{repo}/git/refs/tags --paginate -q '.[].ref' | sed 's|refs/tags/||' | grep -E '^v[0-9]' | sort -V | tail -1
   ```
   If the output is empty, note "(no prior version tag)".
   Record `<default-branch>` via
   `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
3. Draft release notes to `<run-dir>/release-notes.md` using `write_file`:
   - Resolve the post-merge remote tip SHA now:
     `gh api repos/{owner}/{repo}/git/ref/heads/<default-branch> -q .object.sha`
     → record as `<resolved-sha>`.
     **`<resolved-sha>` must be the post-merge remote tip**, not the local HEAD
     (which may be stale), so that `--target` in step 7 and the changelog URL in
     the release notes point at the correct commit.
   - Heading: `## What's Changed`
   - One bullet per merged PR: `- <title> (#<N>) @<author>`
   - If a prior tag exists, footer:
     `**Full Changelog**: https://github.com/<repo>/compare/<prior-tag>...<resolved-sha>`
   (Using the resolved SHA rather than `HEAD` prevents the URL from drifting if
   commits land on the default branch between note-drafting and release creation.)
4. Preview to the operator:

   ```
   ## Release Preview

   Latest tag: <prior-tag> (or "none")
   Merged this session: #101, #103, #112

   ### Draft release notes:
   <contents of release-notes.md>

   Enter the tag for this release (e.g. v1.5.0), or "skip":
   ```

5. Prompt the operator for the tag via `ask_question` (type: text). If the
   operator replies "skip", the question is declined, or the surface is
   non-interactive, skip release creation and note "release: skipped" in the
   terminal summary. **This gate is mandatory** — `--auto-merge` does not
   bypass it because tags are permanent and visible to downstream consumers.
6. Verify the tag does not already exist on GitHub (as a release OR a bare
   git tag):
   ```bash
   GH_ERR=$(gh api repos/{owner}/{repo}/git/refs/tags/<tag> --silent 2>&1 >/dev/null); TAG_STATUS=$?
   # --silent suppresses response body only; stderr is captured into GH_ERR
   # TAG_STATUS: 0 = ref exists; non-zero + "HTTP 404" in GH_ERR = tag absent; non-zero otherwise = API/network error
   ```
   If `TAG_STATUS` is 0 (ref found), inform the operator and re-prompt once.
   On second collision, skip with "release: tag exists".
   If `TAG_STATUS` is non-zero and `GH_ERR` does not contain "HTTP 404",
   surface `GH_ERR` to the operator and re-prompt (do not silently treat as
   "tag absent" — this is an API or network error, not a missing tag).
   (Using `gh api .../git/refs/tags/` queries the underlying git ref, catching
   both bare tags pushed by CI and tags with an associated GitHub Release object.)
7. Create the release:
   `gh release create <tag> --title "<tag>" --notes-file <run-dir>/release-notes.md --target <resolved-sha>`
   If `gh release create` exits non-zero, surface the error output to the
   operator and note "release: failed" in the terminal summary. Do not retry.
8. Record in the terminal summary: `Release: <tag> (<release-url>)`.

If `--release` is set but no PRs were merged, note "release: skipped (no merges)"
in the terminal summary.

### Terminal State

Report Done with a structured summary:

```
## PR Triage Complete — <repo>

| Action | PRs | Result |
|--------|-----|--------|
| Merged | #101, #103 | ✅ |
| Fixed + pushed | #102, #104 | ✅ (re-review: GREEN) |
| Issue (advisory) | #101, #103 | gh#552, gh#553 |
| Skipped | — | — |
| Deferred (pre-filter) | #108 | Not reviewed |
| Still blocked | #107 | Needs upstream API change |
| Release | — | <tag> (<url>) / skipped / not requested |

Session closed <N> of <M> PRs. Pre-filter: <on: X light, Y deferred | off: reason>.
```

Delete the run dir after reporting: `rm -r <run-dir>`, using the literal path
recorded at run start. Only proceed with deletion if the path starts with
`.afk/tmp/pr-triage-`; never delete `.afk/tmp/` itself or any other run's
directory. This cleanup runs regardless of whether Wave 0.5 ran.
