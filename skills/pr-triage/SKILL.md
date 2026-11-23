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
  - STALE_CI: "A PR's last green CI run was performed against a base commit that predates the current tip of origin/<base>. The check passed on a tree that no longer exists, so it cannot guarantee the PR is clean on the real base. Mitigation: Wave 2A step 1b sub-step 2a checks git merge-base --is-ancestor origin/<base> refs/pr-sim/<N> for every GREEN PR; if the branch does not contain the current base and no ratchet/baseline path was matched, the PR moves to SKIP with instructions to update the branch and re-run CI. When a ratchet/baseline path was matched the existing branch-update path (sub-steps 3-5) handles the refresh."
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
   - `trivial` (check): "Does this pull request change only documentation,
     comments, formatting, lockfiles, or dependency version numbers, with no
     change to executable logic, configuration behavior, tests, or files that
     drive agent or tool behavior (such as skill definitions, agent prompts,
     command files, hook configurations, or plugin manifests)?"
   - `ready` (check): "Does this pull request look like finished work that is
     ready for review, with no WIP or draft markers, placeholder code, or TODOs
     describing unfinished parts of the change?"
   - `risk` (score), levels lowest first:
     0. "Touches no security-sensitive or widely shared code."
     1. "Touches internal code shared by several modules."
     2. "Touches a public API, persistence or migrations, or concurrency."
     3. "Touches authentication, credentials, permissions, secret handling, or payments."
3. Before routing, run a deterministic instruction-file check for every PR
   (without sending anything to Jev):

   ```bash
   gh pr diff <N> --repo <repo> --name-only
   ```

   If this command exits with a nonzero status for a PR, mark that PR
   `override: instruction-file` and route it Full instead of Light, subject to
   the Deferred precedence in step 4.

   A PR is an **instruction-file PR** if any returned path matches a rule below.
   All matching is case insensitive; directory rules match any path segment:

   | Matching rule | Examples |
   |---------|----------|
   | `SKILL.md` (any directory) | `skills/pr-triage/SKILL.md` |
   | `AFK.md`, `CLAUDE.md`, `AGENTS.md` (any directory) | `AFK.md`, `docs/CLAUDE.md` |
   | Any directory segment named `skills`, `agents`, or `commands` | `skills/review/SKILL.md`, `plugins/x/agents/reviewer.md` |
   | Any directory segment named `hooks`, `prompts`, or `.claude` | `hooks/pre-tool.md`, `.claude/commands/review.md`, `.claude/settings.json` |
   | `plugin.json` directly inside any `.claude-plugin` directory, or `plugin.json` at repository root | `.claude-plugin/plugin.json`, `plugin.json` |

   In a prompt-only repository these files are the program; a diff that only
   touches them scores as trivial to Jev but requires a Full review.
   Mark every instruction-file PR with `override: instruction-file`.

   **This override only has an effect when the prefilter runs.** Otherwise,
   every PR already goes Full. When it runs, the override replaces only the
   `trivial` verdict for routing purposes; it does not replace the `ready`
   verdict or the Deferred rule.

4. Route each PR:

   | Route | Rule | Override | Wave 1 treatment |
   |-------|------|----------|------------------|
   | **Light** | `trivial` verdict is yes AND `risk` score < 1.0 | None | `skill review <pr-url> --light`, model `sonnet` |
   | **Deferred** | (`isDraft` is true OR `ready` verdict is no) AND the PR was not named in `--prs` | Applies even with `override: instruction-file` | Not reviewed; lands in SKIP with reason `deferred` |
   | **Full** | Everything else, including every PR whose item errored | `instruction-file`: forced Full regardless of `trivial` verdict | Unchanged: `skill review <pr-url>`, model `claude-opus-4-6` |

   If a PR matches both Light and Deferred, Deferred wins.
   If a PR carries `override: instruction-file`, Full wins over Light (Deferred
   still wins over Full for draft or not-ready PRs not named in `--prs`).
5. **Fail open.** If the Jev call errors, the server refuses the paths, or an
   item returns an `error` (for example a diff over Jev's size limit), that PR
   goes Full. If the step 3 path check exits with a nonzero status, mark that PR
   `override: instruction-file` so it goes Full instead of Light, subject to
   the Deferred precedence in step 4. Never retry by passing the diff as `text`:
   that pulls it into context, which is what this wave exists to avoid.
6. Print the routing table under the manifest, then continue to Wave 1 without
   waiting. Routing is read-only, and the operator can pull any deferred PR back
   in at Wave 1.5.

```
## Pre-filter (Jev)

| # | Route | P(trivial) | P(ready) | Risk (0-3) | Override |
|---|-------|------------|----------|------------|----------|
| 101 | Light | 0.93 | 0.97 | 0.2 | — |
| 102 | Full | 0.04 | 0.91 | 2.6 | — |
| 105 | Full | 0.91 | 0.95 | 0.1 | instruction-file |
| 108 | Deferred | 0.01 | 0.12 | 1.1 | — |
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
preflight. Keep the `refs/pr-sim/*` refs until the merge loop below has
finished, because step 1 reuses `refs/pr-sim/<N>` to classify conflicts. Delete
them only when the merge loop has ended for good: after the last PR in the order
has been handled (merged, skipped or moved to BLOCKED), or when the run ends
while the loop is stopped and not resumed:
`git for-each-ref --format='delete %(refname)' refs/pr-sim/ | git update-ref --stdin`.

**Before the first merge**, check that the base branch is green:
`gh pr view <first-N> --json baseRefName,baseRefOid` to get `<base>` and
`<base_sha>`, then `gh api repos/{owner}/{repo}/commits/<base_sha>/check-runs`.
Also read
`gh api repos/{owner}/{repo}/commits/<base_sha>/status --jq '{state, total_count}'`.
If either API read fails, stop the merge loop with the error text; do not merge.
Record the sum of check runs `total_count` and status `total_count` as
`base_check_count`. Initialize `merged_paths` to an empty set before the loop.
When status `total_count` is above 0, treat `state` of `failure` or `error` as
failing and `pending` as pending in the rule below. Ignore status `state` when
its `total_count` is 0.
- `base_check_count` is 0: the base has no check runs or commit statuses, so the
  repo has no CI on this branch. Treat the base as green and proceed. Do not wait
  for checks to appear. Record once in the terminal summary: "base treated as
  green: no check runs or commit statuses at <base_sha7>". Do not repeat this
  note after each merge.
- Otherwise, if any required workflow on `<base_sha>` is failing or still
  pending, or the combined status is failing or pending by the rule above,
  **stop the merge loop** and report: "Base branch <base> is red or
  pending at <base_sha7>; not merging until it is green." Resume only after the
  operator confirms the base is clean.

For each GREEN PR in that order, **sequentially** (never parallel):

1. Refresh the local base ref with `git fetch origin <base>`. A sibling merge
   earlier in this loop moved the remote base, and every later `origin/<base>`
   in this iteration must see it. Then re-check state and mergeability:
   `gh pr view <N> --json state,headRefOid,mergeable,mergeStateStatus`.
   - `state` is not `OPEN` → SKIP ("merged/closed by another actor"); **continue**
     to the next PR instead of stopping the merge loop.
   - `headRefOid` differs from `fetched_sha` → do not merge automatically.
     Inform the operator: "#<N> head moved (<old_sha7> → <new_sha7>) after
     review; the new commits are unreviewed. To proceed safely, re-triage."
     **Gate:** if the operator explicitly names the new SHA and approves merging
     it unreviewed, refresh the local ref with
     `git fetch origin +pull/<N>/head:refs/pr-sim/<N>` and read `headRefOid`
     again. Require both it and `git rev-parse refs/pr-sim/<N>` to equal the
     approved `<new_sha>`; on failure or another move, SKIP. On success set
     `fetched_sha` to `<new_sha>` and continue to step 1a, not directly to merge.
     Keep "merged unreviewed SHA <new_sha7>" for the terminal summary if merged.
     This approval accepts only the head change; step 2 still confirms the final
     merge message and PR body edit. `--auto-merge` never bypasses this head
     approval. Allow this exception only once per PR per run, including entries
     from step 1b sub-step 4; a second head change means SKIP and re-triage.
     Without explicit approval, move to SKIP.
   - `mergeable` is `UNKNOWN` → GitHub is still computing mergeability after a
     sibling merge. Poll: re-run `gh pr view <N> --json mergeable` every 10 s,
     up to 3 minutes (18 attempts). If `mergeable` resolves to `MERGEABLE`,
     continue. If it resolves to `CONFLICTING`, apply the conflict rule below. If
     it has not resolved after 3 minutes, SKIP with reason "mergeable UNKNOWN
     after 3 min; re-run pr-triage to retry."
   - Not mergeable (`CONFLICTING`) — classify the conflict before routing:
     To discover which paths conflict, run:
     `git merge-tree --write-tree --name-only origin/<base> refs/pr-sim/<N>`
     This exits 1 on conflict and prints the tree OID on the first line followed
     by one conflicted path per line, up to the first blank line.
     - **Lockfile or generated-file conflict only** (every conflicted path matches
       a known lockfile or generated pattern: `package-lock.json`, `yarn.lock`,
       `pnpm-lock.yaml`, `*.lock`, or any file listed under `linguist-generated`
       in `.gitattributes`): under `--auto-merge`, treat as a fix-and-re-verify
       item. Dispatch a single `/fix-pr <N>` agent (after 2A completes, alongside
       other BLOCKED PRs in Wave 2B) with the instruction: "resolve the lockfile
       or generated-file conflict by rebasing onto the current base and
       regenerating (check AFK.md / CLAUDE.md / CONTRIBUTING for a regenerate
       command; agent-afk uses `pnpm regen:generated`); do not hand-edit the
       generated content." This `/fix-pr` dispatch pushes to the PR branch
       without a per-PR gate: it inherits the `--auto-merge` bypass, the same
       as every Wave 2B `/fix-pr` dispatch under `--auto-merge`. Without
       `--auto-merge`, move to BLOCKED and inform the operator with the same
       fix guidance.
     - **Semantic conflict** (any conflicted path is not a lockfile or generated
       file): move to BLOCKED bucket and inform operator.
1a. **Closing-keyword check.** A PR can say `Refs #I` in its body because it
   only partly fixes issue `#I` while one of its commits still says
   `Closes #I`. A default merge carries that text onto the base and GitHub
   closes the issue. Before every merge:
   1. List the closing references in the PR title, body and every commit message:
      `gh pr view <N> --json title,body,commits --jq '.title, .body, (.commits[] | .messageHeadline, .messageBody)' > <run-dir>/closing-scan-<N>.txt`
      Capture stderr and exit status separately. If this read exits nonzero,
      do not merge; move the PR to BLOCKED with the error text. Only after a
      successful read, scan the saved file:
      `grep -Eio '\b(close[sd]?|fix(e[sd])?|resolve[sd]?):?[[:space:]]+([[:alnum:]_.-]+/[[:alnum:]_.-]+#|https://github\.com/[[:alnum:]_.-]+/[[:alnum:]_.-]+/issues/|#)[0-9]+' <run-dir>/closing-scan-<N>.txt`
      Grep exit 1 means no matches; any other nonzero exit means BLOCKED with
      the error text, not permission to merge. Also read
      `gh pr view <N> --json closingIssuesReferences --jq '[.closingIssuesReferences[].number]'`,
      which lists issues linked through the PR body or the Development sidebar
      but not issues named only in commit messages. For this read and every
      `gh issue view` and `allow_squash_merge` read below, capture stderr and
      exit status; on nonzero exit or malformed output, do not merge and move
      the PR to BLOCKED with the error text. Ignore references that name a
      different repository.
   2. For each issue `#I` found, decide whether this PR fully addresses it.
      Read `gh issue view <I> --json title,body` and compare it with the PR
      body, the Wave 1 review result and the diff. Count `#I` as **partial**
      if any of these holds: the PR body mentions `#I` only with a non-closing
      word (`Refs`, `Part of`, `Related to`) while the title or a commit closes
      it; the PR body or the review says part of `#I` is deferred, left out or out of
      scope; `#I` lists separate items and the diff does not address every
      one. When the evidence is unclear, count it as partial: an issue left
      open by mistake is cheap to close by hand, while a wrongly closed one
      hides unfinished work.
   3. No partial issues: step 3 uses the default message. Continue to step 1b,
      except an accepted update rescan returns to step 1b sub-step 5.
   4. One or more partial issues: never use the default merge message. Read
      `gh api repos/{owner}/{repo} --jq .allow_squash_merge` under the fail
      closed read rule above, then build an explicit message:
      - Write the subject to `<run-dir>/merge-subject-<N>.txt`, starting from
        `<PR title> (#<N>)`. Rewrite every closing reference to a partial issue
        as `Refs #I`, preserving a sensible description and the PR number suffix.
        For example, `Fix #12: handle empty input (#34)` becomes
        `Handle empty input (Refs #12) (#34)`. Fully addressed references stay.
      - Write the body to `<run-dir>/merge-body-<N>.md`: the PR body, with every
        closing reference to a partial issue rewritten to `Refs #I`. References
        to fully addressed issues stay as they are. Add any other operator merge
        rule (for example a co-author trailer) to this same file.
      - Before step 2, re-run the `grep -Eio` pattern from sub-step 1 on both
        the subject file and the body file. Confirm neither contains a closing
        reference to a partial issue; otherwise correct the files before proceeding.
        Apply the same grep exit status rules from sub-step 1 to both file scans.
      - If a closing reference to a partial issue appears in the **PR body**,
        also write the rewritten body to `<run-dir>/pr-body-<N>.md` and, at
        merge time, run `gh pr edit <N> --body-file <run-dir>/pr-body-<N>.md`
        before the merge command. GitHub closes issues linked from the PR body
        no matter what the merge commit says. This edit shares the step 2 merge
        gate, including its `--auto-merge` bypass and Light route exception.
        If `gh pr edit` exits nonzero, do not merge; move the PR to BLOCKED with
        the error text.
      - If a closing reference to a partial issue appears in a **commit
        message**, the merge must use `--squash`: `--merge` and `--rebase` put
        the original commits on the base unchanged, so their keywords still
        close the issue. If `allow_squash_merge` is `false`, do not merge:
        move the PR to SKIP with reason
        "commit closes partial issue #I and squash merge is disabled; reword
        the commit, then re-run", and inform the operator.
      - If a partial issue is in `closingIssuesReferences` but no closing
        keyword for it appears in the PR body, it is linked through the
        Development sidebar, which `gh pr edit` cannot remove. Move the PR to
        SKIP with reason "issue #I is linked to close on merge; unlink it on
        GitHub, then re-run", and inform the operator.
   **Gate:** the rewritten message is part of the merge decision. Without
   `--auto-merge` (and always for Light route PRs), step 2 shows the rewritten
   subject, the full body file, the list of rewritten references and any PR body edit
   in the merge prompt. Under `--auto-merge`, record the same in the terminal
   summary for that PR: "#<N> merged with rewritten message: <rewritten
   references>; PR body edited: yes/no", followed by the rewritten subject and
   body file contents.
1b. **Fresh-base CI for ratchet and baseline files.** A PR whose checks passed
   on an older base can still break the base when it merges, if it touches a
   file whose correct content depends on the base (a ratchet count, a
   generated baseline, a snapshot) or a file a sibling merged earlier in this
   run. Such a PR needs a CI run on the current base before it merges.
   1. List the PR's changed paths:
      `git diff --name-only origin/<base>...refs/pr-sim/<N>`.
   2. A path is a **ratchet or baseline path** if either row below matches:

      | Source | Matches when |
      |--------|--------------|
      | Repo docs | The path is listed under `linguist-generated` in `.gitattributes`, or AFK.md, CLAUDE.md or CONTRIBUTING names it (or a directory or glob containing it) as generated, ratchet or baseline content, or as the output of a documented regenerate or baseline-update command (the same search Wave 2B uses). |
      | This run | The path is in `merged_paths`, the set of paths changed by PRs merged earlier in this run (step 4 records it; empty before the first merge). |

      If no path matches, go to sub-step 2a.
   2a. **CI-freshness check (all PRs).** Regardless of whether any ratchet or
       baseline path was matched, verify that the PR's last CI run was performed
       on top of the current base:
       ```
       git merge-base --is-ancestor origin/<base> refs/pr-sim/<N>
       ```
       - Exit 0 → the PR branch already contains the current base tip. If no
         ratchet/baseline path was matched, skip to step 2. If a ratchet/baseline
         path was matched, continue to sub-step 3.
       - Exit 1 → the PR branch is **behind** `origin/<base>`: its last green CI
         run predates commits that have since landed on the base. The check passed
         on a tree that no longer exists.
         - If a ratchet/baseline path was matched: continue to sub-step 3
           (the branch-update path already handles the refresh).
         - Otherwise: move this PR to **SKIP** with reason "CI is stale — branch
           is behind origin/<base> (<base_sha7>); update the branch
           (`gh pr update-branch <N>`) and wait for a fresh green CI run before
           merging. Refs #5079." Do not attempt merge; continue to the next PR.
       - Any other exit or command failure → BLOCKED with the error text; never
         merge on an unreadable result.
   3. First, if `base_check_count` is 0, go directly to step 2 without updating
      the branch. Record "#<N> touches <matched paths>; no CI in repo, merged
      without a fresh-base run" in the terminal summary only if merged.
      Otherwise, if `git merge-base --is-ancestor origin/<base> refs/pr-sim/<N>` exits 0,
      the branch already contains the current base: skip to sub-step 5.
      Otherwise the branch must be updated.
   4. Update the branch with `gh pr update-branch <N>` only after the gate below.
      This pushes a merge commit to the PR branch. Without `--auto-merge` (and
      always for Light route PRs), ask the operator first:
      `"#<N> touches <matched paths> and is behind <base>. Update its branch with <base> (pushes a merge commit) and wait for CI before merging? (y/n)"`
      Under `--auto-merge` the update runs without this prompt: it inherits the
      `--auto-merge` bypass, the same as the Wave 2B `/fix-pr` pushes. Record
      "#<N> branch updated with <base> at <current_base_sha7>" in the terminal
      summary only after a successful update, taking `<current_base_sha7>` from
      `git rev-parse --short origin/<base>` in this iteration.
      If the operator declines, move the PR to SKIP with reason "needs a CI run
      on the current base". After approval (or its permitted bypass), immediately
      before `gh pr update-branch <N>`, run `git fetch origin <base>`, record
      `expected_base_sha` from `git rev-parse origin/<base>`, and record
      `expected_tree` from
      `git merge-tree --write-tree <fetched_sha> <expected_base_sha>`.
      On any of these command failures, do not update and apply the "head moved"
      rule from step 1 as qualified below. If the update command fails (for
      example a conflict, or a fork that does not allow maintainer edits), move
      the PR to BLOCKED with the error text.
      After a successful update, read `new_head` from
      `gh pr view <N> --json headRefOid`, its parents from
      `gh api repos/{owner}/{repo}/commits/<new_head> --jq '[.parents[].sha]'`,
      and its tree from
      `gh api repos/{owner}/{repo}/commits/<new_head> --jq .commit.tree.sha`.
      Require exactly two parents in order: `<fetched_sha>` then
      `expected_base_sha`, and require the tree to equal
      `expected_tree`. Refresh the local ref with
      `git fetch origin +pull/<N>/head:refs/pr-sim/<N>` and require
      `git rev-parse refs/pr-sim/<N>` to equal `new_head` too. Only when all
      checks succeed, set `fetched_sha` to `new_head`, then re-run step 1a once
      using fresh reads so the scan includes the new commit message. Discard
      prior step 1a decisions and regenerate any message or PR body files from
      these reads. Apply all step 1a rules, including BLOCKED and SKIP outcomes.
      If still eligible, continue directly to sub-step 5, not back to step 1b
      sub-step 1. This rescan happens once per accepted update and never triggers
      another branch update by itself.
      On any parent, tree or local ref mismatch or failure reading the new head,
      parents, tree or local ref, apply the "head moved" rule from step 1.
      For any such entry (including a failed tree computation),
      first read the current `headRefOid` as `<new_sha>`; if that read fails or
      it still equals `fetched_sha`, SKIP with the verification error rather than
      requesting approval. An approved exception restarts at step 1a once, so
      all message and CI checks use the approved head.
   5. Poll `gh pr checks <N> --json name,bucket` immediately, then at a fixed
      interval under two separate limits. While no check is listed yet, poll
      every 10 s for at most 18 attempts. Once any check is listed, poll every
      30 s for at most 60 further attempts, because a CI run needs longer to
      finish than to appear. Capture stdout, stderr and exit status even
      when nonzero; do not let a shell error setting abort the loop. Pending
      checks exit 8, failures exit nonzero, and no checks produces an error.
      Read valid JSON buckets rather than interpreting every nonzero exit as
      failure. Before accepting a passing result, re-read `headRefOid` with
      `gh pr view <N> --json headRefOid`; if it differs from `fetched_sha`,
      apply the "head moved" rule from step 1; a read failure means BLOCKED.
      - A nonempty list containing only `pass` or `skipping`: proceed to step 2.
      - Any `fail` or `cancel`: move to BLOCKED with reason "checks failed on
        current base at <fetched_sha>" and the failing or cancelled check names.
      - An empty list or the explicit no checks error: retry within the 18
        attempt limit. At the limit, SKIP with reason "no checks appeared on
        <fetched_sha> after 18 attempts" and go to the next PR.
      - Any `pending` with no `fail` or `cancel`: retry within the 60 attempt
        limit. At the limit, SKIP with reason "checks still pending on
        <fetched_sha> after 60 attempts" and go to the next PR.
      - Any other error, malformed JSON or unknown bucket: BLOCKED with the
        error text; never merge on an unreadable result.
2. Unless `--auto-merge` is set, confirm with operator:
   `"Merge #<N> '<title>' into <base>? (y/n)"`
   When step 1a rewrote the message, use this prompt instead, followed by the
   rewritten subject, the body file contents and the PR body edit (if any):
   `"Merge #<N> '<rewritten subject from merge-subject file>' into <base> with a rewritten message (closing keywords for partial issues <#I list> changed to Refs)? (y/n)"`
   `--auto-merge` does not cover PRs on the Light route: always confirm those
   individually, since they were reviewed on a cheaper model.
3. Merge: `gh pr merge <N> --merge --delete-branch --match-head-commit <fetched_sha>`
   (Use `--merge` not `--squash` unless the PR is single-commit; respect repo's
   merge strategy if detectable from `gh api repos/{owner}/{repo}` settings.)
   When step 1a rewrote the message, run the `gh pr edit` from step 1a first if
   it applies, then merge with the explicit subject and body file:
   `gh pr merge <N> --squash --subject "$(cat <run-dir>/merge-subject-<N>.txt)" --body-file <run-dir>/merge-body-<N>.md --delete-branch --match-head-commit <fetched_sha>`
   Use `--merge` in place of `--squash` in that same command, keeping the explicit
   subject and body file, when squash is disabled and no commit message holds a
   closing reference to a partial issue, or when no such commit reference exists
   and the repo strategy calls for merge commits.
   If `gh pr merge` exits nonzero, move the PR to BLOCKED with the error text
   and note any PR body edit already made in the terminal summary. Do not run
   step 4 for a failed merge; continue to the next PR.
   **`--match-head-commit` is the last-resort guard against merge races.** If a
   concurrent pr-triage run (or any other actor) merges this PR or pushes new
   commits between step 1's state check and this merge call, GitHub will reject
   the merge because the head SHA no longer matches `<fetched_sha>`. This
   eliminates the risk of landing an unreviewed commit. The TOCTOU window between
   step 1 and this call is accepted as irreducible without repo-side locking;
   `--match-head-commit` is the hard backstop that makes it safe.
4. **After each merge**, add the PR's changed paths from step 1b sub-step 1
   to `merged_paths`. Then verify the base branch CI before proceeding:
   First, obtain and record `new_base_sha`:
   `gh api repos/{owner}/{repo}/git/ref/heads/<base> --jq .object.sha`
   Wait 5 s for CI to pick up the new base SHA, then check
   `gh api repos/{owner}/{repo}/commits/<new_base_sha>/check-runs` and
   `gh api repos/{owner}/{repo}/commits/<new_base_sha>/status --jq '{state, total_count}'`.
   On either read failure, stop the merge loop with the error text; do not merge
   further. For each read or retry, compute `new_base_check_count` as the sum
   of both `total_count` values. When status `total_count` is above 0, treat
   `state` of `failure` or `error` as failing and `pending` as pending in the
   rule below; ignore status `state` when its `total_count` is 0.
   - `new_base_check_count` is 0 and `base_check_count` is 0: the repo has no CI.
     Treat the new base as green and proceed to the next PR.
   - `new_base_check_count` is 0 but `base_check_count` was above 0: CI has not
     registered the new SHA yet. Read both endpoints again every 10 s, up to
     3 minutes (18 attempts). If the sum becomes positive, apply the rule below.
     If it is still 0 at the limit, **stop the merge loop** and inform the
     operator: "No check runs or commit statuses appeared on <base> at
     <new_base_sha7> after merging #<N>; not merging further until CI reports."
     Resume only after the operator confirms CI has reported on the base.
   - Otherwise, if any required workflow is failing or still pending on the new
     base SHA, or the combined status is failing or pending by the rule above,
     **stop the merge loop** and inform the operator: "Base branch
     <base> is red or pending at <new_base_sha7> after merging #<N>; not merging
     further until it is green." Resume only after the operator confirms the
     base is clean.
   After a green result, set `base_check_count` to `new_base_check_count` before
   proceeding to the next PR.

Whenever the merge loop stops, retain PRs not yet attempted for a possible
resume. If the run ends without resuming, list them as not attempted in the
terminal summary; do not silently count them as merged, SKIPped or BLOCKED.

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
- **Labels:** apply the first label the repo has in this order: `review-followup`, `follow-up`, `tech-debt`.

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
   (`<resolved-sha>` is snapshotted here at draft time; the drift window lasts
   however long the operator review takes and is accepted. If commits land on
   the default branch during steps 4 and 5, only the changelog URL is affected
   by that drift because the release itself is pinned to the reviewed commit
   via `--target <resolved-sha>` in step 7. Resolving the SHA again immediately
   before step 7 would risk releasing commits the operator never reviewed and
   that the drafted notes do not describe.)
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

Notes:
1. Rewritten merge messages: <PR, rewritten references, subject and full body contents, or none>.
2. PR body edits: <PR, yes/no; include edits already made when a merge failed>.
3. Branch updates: <PR, base and base SHA, or none>.
4. Merges without a fresh-base run: <PR and matched paths, or none>.
5. Base treated as green with zero check runs and commit statuses: <base SHA, or none>.
6. Merged unreviewed SHA: <PR and approved SHA, or none>.
7. PRs not attempted after a stopped merge loop: <PRs and stop reason, or none>.

Session closed <N> of <M> PRs. Pre-filter: <on: X light, Y deferred | off: reason>.
```

Delete the run dir after reporting: `rm -r <run-dir>`, using the literal path
recorded at run start. Only proceed with deletion if the path starts with
`.afk/tmp/pr-triage-`; never delete `.afk/tmp/` itself or any other run's
directory. This cleanup runs regardless of whether Wave 0.5 ran.
