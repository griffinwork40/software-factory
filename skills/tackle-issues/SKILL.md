---
name: tackle-issues
description: "Tackle N open GitHub issues in parallel using worktree-isolated subagents, producing one PR per issue. Lightweight wrapper — PRs are proposals, not mutations; the safety net is /pr-triage on the next pass."
argument-hint: "<N> [--label <label>] [--skip <glob>] [--issues <N,N,...>] [--repo <owner/repo>] [--draft] [--sort oldest|newest]"
category: "Build & ship"
when-to-use: "When you have open issues to burn down in parallel. Triggers on 'tackle issues', 'work on open issues', 'burn down issues', 'make PRs for issues'."
flags: [--label, --skip, --issues, --repo, --draft, --sort]
max_tool_use_iterations: 100
failure_modes:
  - STALE_BASE: "Subagent worktree HEAD is not a descendant of the fetched origin HEAD (or the SHA is unresolvable). Mitigation: subagent step 1 runs git merge-base --is-ancestor and reports STALE-BASE on any non-zero exit."
  - ALREADY_DONE: "Issue already addressed by merged commits or PRs but remains open. Mitigation: step 1 verifies each candidate against HEAD via git log and gh pr list before dispatch."
  - DUPLICATE_PR: "An open PR already exists for the issue. Mitigation: step 1 checks closingIssuesReferences and searches open PRs by issue number."
  - WRONG_CHECKOUT: "--repo names a different repository than the cwd checkout, so worktrees branch from the wrong repo. Mitigation: none yet (tracked in #3)."
---

# Tackle Issues

Dispatch parallel subagents in worktrees to tackle open GitHub issues and make PRs.

## Arguments

Parse from the input:

| Arg | Default | Meaning |
|-----|---------|---------|
| `<N>` (positional) | 5 | How many issues to tackle |
| `--issues 201,203,207` | (auto-discover) | Specific issue numbers |
| `--label <label>` | (none) | Filter issues by GitHub label |
| `--skip <glob>` | (none) | Skip issues whose title matches this pattern |
| `--repo owner/repo` | (cwd git remote) | Target repository |
| `--draft` | off | Create draft PRs instead of ready-for-review |
| `--sort oldest\|newest` | newest | Issue selection order. `newest` (default) tackles recently opened issues first; `oldest` clears the backlog |

## Execution

### Step 1 — Discover issues

Run `git fetch origin` once. Record:
- `<default-branch>` via `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`
- `<head-sha>` via `git rev-parse origin/<default-branch>`

This fetch also determines where isolated subagent worktrees start: the runtime bases `isolation: "worktree"` on the cached remote default ref, so after the fetch every subagent worktree will begin at `<head-sha>`.

If `--issues` provided, fetch those issue records directly with `gh issue view <n> --json number,title,body,labels,assignees` for each. **Do not apply `--label`, `--skip`, or assignee filters to explicit `--issues`; those flags apply only to auto-discovered issues.** Otherwise:

```bash
gh issue list --state open --limit <N * 2> --sort created --order <asc if --sort oldest, desc otherwise> --json number,title,body,labels,assignees
```

Filter out issues that (auto-discover only):
- Are assigned to someone else
- Match `--skip` pattern
- Don't match `--label` filter (if set)
- Already have an open PR linked. Check via `gh issue view <number> --json closedByPullRequestsReferences` (linked PRs) plus `gh pr list --state open --search "#<number> in:title,body"` and scan open PRs whose branch name suggests the issue number. If any linked or referencing open PR exists, skip.

Before selecting issues to dispatch, verify **every candidate** (including explicit `--issues`) against `<head-sha>`. For each candidate issue `<N>`:

1. Check open referencing PRs and issue comments (as above).
2. Run `git log origin/<default-branch> --oneline -E --grep "#<N>([^0-9]|$)"` to find commits that reference the issue (the `([^0-9]|$)` suffix prevents `#20` from matching `#200`; do not use `\b`, which POSIX ERE does not support, so git silently matches nothing; any match still warrants inspection).
3. Run `gh pr list --state merged --search "#<N>" --json number,title,body,closingIssuesReferences` to find merged PRs that mention the issue. Prefer `closingIssuesReferences` for an authoritative match; fall back to scanning titles and bodies for an exact `#<N>` reference (GitHub full-text search tokenizes `#20` loosely, so a keyword hit alone is not conclusive).

If any merged commits or merged PRs exist for a candidate, inspect the issue's remaining acceptance items against HEAD. If nothing remains unaddressed, skip and report as **already-done**. If partial work was merged but items remain, note the remaining items and include the candidate (the subagent will address only what is left).

Take the first `<N>` issues that pass filters and this verification.

If 0 issues pass, report Done with "No eligible issues found" and stop.

### Step 2 — Present the hit list

Show the operator what's about to happen:

```
## Tackling <N> issues — <repo>

| # | Title | Labels |
|---|-------|--------|
| 201 | Add null check in auth.ts:47 | bug, low |
| 203 | Remove unused import in config.ts | cleanup |
| 207 | Add test for edge case in parser | test-gap |

Dispatching <N> parallel subagents in worktrees...
```

If any candidates were skipped during selection, append a "Skipped at selection" table:

```
### Skipped at selection

| # | Title | Reason |
|---|-------|--------|
| 198 | Fix typo in README | Already fixed on <head-sha> |
| 200 | Update auth flow | Claimed by open PR #211 |
| 205 | Add retry logic | Already done — merged in PR #198, PR #201 |
```

### Step 3 — Parallel dispatch

Dispatch one `agent` tool call per issue. Issue all calls for the current wave in a **single reply** so they run in parallel.

- `agent_type: "general-purpose"`
- `isolation: "worktree"` — each agent gets its own worktree; dirty/ahead worktrees are preserved and locked by the runtime; clean ones are torn down automatically
- `model: "claude-sonnet-4-6"`
- Cap at **5 concurrent**. If there are more than 5 issues, batch them into waves of 5: dispatch wave 1 (5 calls in one reply), wait for all 5 to complete, then before dispatching each subsequent wave re-check that remaining candidate issues are still unclaimed (no new open PRs referencing them since the last wave). For each pending candidate run these two recheck calls in parallel (issue both in a single reply to halve inter-wave latency): `gh pr list --state open --search "#<N> in:title,body"` **and** `gh issue view <N> --json closingIssuesReferences` (to catch PRs linked via GitHub's auto-close UI that won't appear in a title/body search). Note: the recheck uses `closingIssuesReferences` rather than the `closedByPullRequestsReferences` field used in Step 1 discovery — these are different GitHub fields by design; a PR that opens AND merges within one wave span could slip through both, but that window is temporally negligible and non-data-affecting. If `gh` exits non-zero during either recheck call, log the issue number as **UNCHECKED**; UNCHECKED candidates are eligible for dispatch — treat them as unclaimed and proceed rather than silently skipping them. Then dispatch the next wave.

Each agent's prompt:

```
You are fixing GitHub issue #<number> in <repo>.

**Issue title:** <title>
**Default-branch HEAD:** <head-sha> (<default-branch>)
**Issue body:**
<body>

Instructions:
1. Run `git rev-parse HEAD` and record it as `<worktree-sha>`. Verify the base is reachable: `git merge-base --is-ancestor <head-sha> HEAD`. If the exit code is non-zero (exit 1 means not an ancestor; exit 128 means the SHA is unresolvable — either way, stop without editing any file and report **STALE-BASE** with `<worktree-sha>` and `<head-sha>`). Otherwise, before any edit, verify the issue comments and open referencing PRs have not made the remaining ask obsolete; if obsolete, report **OBSOLETE** and stop.
2. For a **BUG**, establish on current HEAD either a failing reproduction (test or command) or a concrete source-level demonstration (file:line of the defective code path and why it is reachable). If neither is possible, do not fix it; report **UNREPRODUCED** and what you checked.
3. For a **PREVENTIVE/refactor/hygiene** issue, show evidence of the current risk on HEAD (for example, the hand-rolled site still exists at file:line, the gate fails, or the size remains over its ceiling). Cosmetic findings with no current risk are reported, not fixed.
4. For any **other issue type** (feature, test-gap, docs, or anything not covered above), confirm on HEAD that the requested change is not already present: cite the relevant file:line or paste the grep output showing absence. If already present, report **OBSOLETE** and stop.
5. Make the fix only after whichever of steps 2, 3, or 4 applies. Run relevant tests and `pnpm lint` (or the project's lint command) to verify the change compiles.
6. Commit with message: `fix(#<number>): <short description>`
   Write the message to a temp file inside your worktree (not /tmp, which forked
   subagents are denied) using the file-write tool or a quoted heredoc redirected
   to the file. Do not commit the temp file. Then run:
   `git commit -F <tmpfile>`
   Do not: inline the message with `-m`; use `git commit -F - <<'EOF'` stdin
   heredocs; or use `$(cat <<'EOF' ...)` command substitution (macOS bash 3.2
   mis-parses apostrophes inside it, exit 2).
7. Create a PR. Choose the closing keyword carefully:
   - Use `Closes #<number>` when this PR completes **every remaining acceptance item**
     for the issue (including cases where earlier partial PRs used `Refs #<number>`
     and this PR addresses what was left).
   - Use `Refs #<number>` only when this PR covers **partial work** and items remain.
     In that case, immediately post a comment on the issue listing the exact remaining
     items so the next agent can find them. Write the comment body to a temp file
     inside your worktree (not /tmp), then run:
     `gh issue comment <number> --body-file <tmpfile>`
   - For a **tracking/checklist issue**, you may address one unchecked box per PR
     using `Refs #<number>` and naming the checkbox in the PR title and body.
   - Write the PR body to a temp file inside your worktree (not /tmp), then run:
     `gh pr create --title "fix(#<number>): <short description>" --body-file <tmpfile>`
   - PR body must include the chosen closing/refs keyword, a brief description of the
     change, and a **Verified on HEAD** section citing <head-sha> and the reproduction,
     demonstration, or risk evidence.
   - Add `--draft` flag if instructed.
8. Report what you did: PR URL, files changed, tests run.

Do NOT:
- Touch files unrelated to the issue
- Make architectural changes or large refactors
- Combine multiple issues into one PR
- Force push or modify other branches
- Add entries to `.filesize-baseline.json`, `.funcsize-baseline.json`, or similar
  audit baseline files. If a file exceeds its size ceiling, extract one whole
  concern into a new sibling file instead of raising the baseline. Update all
  imports in every file that referenced the extracted symbols, then run
  `pnpm lint` (or the project's lint command) and relevant tests to confirm no
  references are broken before committing.
```

### Step 4 — Report results

After all agent calls complete (all waves if batched), collect each agent's final message and summarize:

```
## Results

| # | Issue | PR | Status | Summary |
|---|-------|----|--------|---------|
| 201 | Add null check in auth.ts:47 | #212 | ✅ | Added guard + test |
| 203 | Remove unused import | #213 | ✅ | Removed 3 unused imports |
| 207 | Add parser test | — | UNREPRODUCED | Couldn't reproduce edge case |
| 209 | Document config options | — | OBSOLETE | Change already present on <head-sha> |
| 210 | Migrate auth helper | — | STALE-BASE | base <head-sha> not ancestor of worktree <worktree-sha> |

Status legend: ✅ PR opened | ❌ failed | UNREPRODUCED (no reproduction found) | OBSOLETE (already done or ask gone) | STALE-BASE (base SHA not an ancestor of worktree HEAD, or unresolvable) | cosmetic (reported, not fixed)

<N>/<M> issues tackled. Run `/pr-triage` to review and merge.
```

Worktree lifecycle is managed automatically by the `agent` tool when `isolation: "worktree"` is used — no manual cleanup needed. Dirty or ahead worktrees are preserved and locked so the operator can inspect; clean worktrees are torn down by the runtime.

### That's it

No merge gates, no human checkpoints on PR creation — PRs are proposals.
The safety net is `/pr-triage` on the next pass, which reviews with full gates.
