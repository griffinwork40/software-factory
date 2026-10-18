---
name: land
description: "Take ONE open PR all the way to merged: review it, fix blocking findings + red CI + merge conflicts, re-review until the blocking set is empty (bounded, convergence-checked), merge the exact SHA that was reviewed, then file every deferred finding as a deduplicated follow-up issue. Chains /review and /fix-pr into a closed loop with a per-finding completeness ledger, so the operator types one verb instead of babysitting review → fix → re-review → 'merge and make issues for any follow-ups'. Emits LANDED / READY / STALLED / BLOCKED / ALREADY-LANDED."
when-to-use: "When a single PR should end up merged and the operator would otherwise drive the review/fix/re-review/merge/follow-up loop by hand. Triggers: 'land 2806', 'get this PR merged', 'review it, fix it, merge it', 'merge and make issues for any follow-ups', '/fix-pr N --re-review' followed by a merge request."
argument-hint: "<PR-number-or-URL> [--repo <path>] [--max-rounds <N=3>] [--no-merge] [--auto-merge] [--no-issues] [--nits] [--dry-run]"
category: "Build & ship"
context: load
skills: [review, fix-pr, contract]
flags: [--repo, --max-rounds, --no-merge, --auto-merge, --no-issues, --nits, --dry-run]
failure_modes:
  - UNREVIEWED_MERGE: "A commit lands on the head branch after the last review and gets merged unseen. Mitigation: merge passes --match-head-commit <reviewed_sha>; GitHub refuses the merge if the head moved."
  - NONCONVERGENT_LOOP: "Each fix introduces a new blocker and the loop never ends. Mitigation: hard round cap, plus a convergence rule — the same blocking defect surviving two fix rounds is STALLED, never a third attempt."
  - LOST_FOLLOWUP: "A non-blocking finding is waived at merge time and never tracked. Mitigation: the per-finding ledger requires every finding to end in exactly one bucket; a finding with no bucket blocks the LANDED verdict."
  - REVIEW_FINDINGS_INVISIBLE_TO_FIX: "/review is read-only and never posts, so its findings exist only in the coordinator's context, while /fix-pr builds its spec from GitHub stores. Mitigation: land posts the blocking set as an issue-level PR comment (gh pr comment), which /fix-pr's Phase 1 source 3 (gh pr view --json comments) already harvests."
  - NESTED_REVIEW_COLLISION: "/review invoked from a subagent hits max_depth. Mitigation: context: load — land runs in the top-level coordinator and calls /review and /fix-pr directly, never from a fork."
  - DUPLICATE_ISSUES: "Follow-ups re-file something already tracked. Mitigation: search open issues per follow-up; comment on the existing issue instead of creating a new one."
---

# land

One PR in, one merged PR plus tracked follow-ups out.

**Why this is a verb.** Without it, the operator drives the loop by hand: `/review`, then `/fix-pr`, then `/review` again, then the merge, then "make issues for any follow-ups". A single PR can take many such turns. Every step already had a skill; nothing closed the loop.

Within this plugin, `/land` is the single-PR counterpart to `/pr-triage`: `/pr-triage` sorts many PRs in one pass, `/land` drives one PR to convergence.

## Sub-agent contract
/contract

## Boundary vs. incumbents

| Skill | What it does | Why it is not /land |
|---|---|---|
| `/review` | Read-only verdict on a diff | Never fixes, posts, or merges |
| `/fix-pr` | One harvest → fix → push pass, optional single re-review | Never loops to convergence, never resolves conflicts, never merges, never files issues |
| `/pr-triage` | Batch triage of 2+ PRs, one fix pass, offer to merge | Batch-shaped; no convergence loop, no conflict resolution, no per-finding ledger |
| `/ship` | Local unpushed work → PR | Starts before a PR exists; /land starts after |

/land composes /review and /fix-pr; it re-implements neither. If you are writing a reviewer prompt or a fix subagent prompt inside this skill, you are in the wrong layer.

## Hard constraints

1. **Coordinator only.** Runs in the top-level session (`context: load`). /review and /fix-pr are invoked directly from here, never from a subagent. If `get_runtime_state` `self.depth` is a number > 0 (top level reports `null`), stop with BLOCKED: "land must run top-level".
2. **Merge only what was reviewed.** `reviewed_sha` = the PR `headRefOid` captured immediately before the final /review is invoked. The merge call carries `--match-head-commit <reviewed_sha>`, so any push after that review invalidates it and GitHub refuses the merge.
3. **No force.** Plain `git push` to the PR head branch only. Conflicts are resolved by merging base into head, never by rebase + force.
4. **Bounded and convergent.** At most `--max-rounds` fix rounds (default 3). A blocking defect that survives two fix rounds means STALLED, not another attempt.
5. **Every finding gets exactly one fate.** `fixed` · `triaged-out` (by /fix-pr probes, with reason) · `waived-in-review` (non-blocking at final review) → `filed #N` or `existing #N` or `skipped-nit` · `out-of-scope` (declared by /fix-pr) → `filed #N` · `unresolved`. Any `unresolved` finding rules out LANDED.
6. **Ledger is the source of truth.** Append to `<repo>/.afk/land/pr-<N>.md` after every phase. On resume, read it first and continue from the last completed phase.

## Arguments

Parse these from `$ARGUMENTS`:

| Flag | Default | Meaning |
|------|---------|---------|
| `<PR-number-or-URL>` | (required) | The PR to land |
| `--repo <path>` | (cwd git root) | Local checkout of the target repo |
| `--max-rounds N` | 3 | Cap on fix rounds before STALLED |
| `--no-merge` | off | Converge, then stop at READY without merging |
| `--auto-merge` | off | Skip the merge confirm when /land was model-routed rather than typed |
| `--no-issues` | off | Skip Phase 3; deferred findings are listed but not filed |
| `--nits` | off | Also file nit-severity findings as follow-ups |
| `--dry-run` | off | Stop after the first review and print planned actions; no posts, pushes, merges, or issues |

## Execution

### Phase 0: Preflight (inline, fail closed)

1. Parse args. `pr` is required; `--repo` defaults to the cwd git root.
2. `gh auth status`; `gh pr view <pr> --json number,title,url,state,isDraft,baseRefName,headRefName,headRefOid,isCrossRepository,mergeable,author,maintainerCanModify`.
   - `state == MERGED` → **ALREADY-LANDED**. If a ledger from an earlier run exists, still run Phase 3 for the follow-ups it holds; with no ledger, stop.
   - `state == CLOSED` → BLOCKED.
   - `isDraft` → BLOCKED ("mark ready first") unless the operator explicitly asked to land a draft.
   - Cross-repo head with `maintainerCanModify == false` → fixes cannot be pushed; continue in review-only mode and stop at READY or BLOCKED.
3. **Merge authority.** The human typing `/land <pr>` is explicit intent to merge. If /land was routed by a model rather than typed by the operator, and `--auto-merge` is absent, the merge gate in Phase 2 requires an `ask_question` confirm. `--no-merge` always stops at READY.
4. **Merge strategy.** `gh api repos/{owner}/{repo} --jq '{squash:.allow_squash_merge,merge:.allow_merge_commit,rebase:.allow_rebase_merge,del:.delete_branch_on_merge}'` plus `git log origin/<base> --format=%s -10`. If base history is `<subject> (#NNNN)` lines and squash is allowed → squash. Otherwise use the first allowed of merge > rebase.
5. **Operator merge rules.** `memory_search "merge"` scoped to this repo, and read hot memory. Collect every rule that touches merging: commit identity or co-author rewrites, required squash body handling, contributor thank-yous. These are applied in Phase 2 and 4, never improvised.
6. Write the ledger header: PR, base, head, strategy, rules collected, flags, start time.

`--dry-run` stops after Phase 1's first review and prints the planned actions, with no posts, pushes, merges, or issues.

### Phase 1: Converge (round loop, `r = 1..max_rounds`)

#### 1a. Sync with base
Re-read `mergeable`. If `UNKNOWN`, poll it (`wait_for` command, 10 s interval, 2 min max). If `CONFLICTING`:
- `worktree create` (name `land-<pr>-sync`, base = head branch after `git fetch`). Never raw `git worktree add`.
- In it: `git merge origin/<base>`. If clean → run the project's lint + test gates → plain push.
- If conflicted → dispatch ONE subagent (`cwd` = the worktree, sonnet, `max_tool_use_iterations: 60`) to resolve, preserving the intent of both sides, and to run the gates. Brief it to resolve code conflicts first, then regenerate generated or baseline files with the repo's documented command if one exists (check AFK.md / CLAUDE.md / CONTRIBUTING / `.gitattributes`; agent-afk has `pnpm regen:generated`) instead of hand-merging them. It must not push. The coordinator then checks `git diff --check`, checks that no `<<<<<<<`/`>>>>>>>` markers remain in tracked files, re-runs the gates itself, commits the merge, and pushes plain.
- Gates red after one resolution attempt → BLOCKED with the worktree kept.
- Ledger: `round r: synced base <sha> → head <sha>`.

#### 1b. Review
Capture `headRefOid` (`gh pr view <pr> --json headRefOid`) as this round's `reviewed_sha`, then invoke `/review <pr>` directly. From round 2 on, pass `--brief` holding the numbered blocking list from the previous round, so the spec-compliance dimension checks those items specifically. Parse:
- the `Decision: MERGE|DO NOT MERGE — <counts>` line (absent → the hop FAILED; retry once, then BLOCKED);
- every finding: severity, `blocking`, dimension, `file:line_range`, `ref:<sha>`, finding sentence, suggestion.
Cross-check: every finding's `ref:<sha>` should equal `reviewed_sha`; a mismatch means the head moved mid-review, so discard the review and repeat 1b (counts against nothing). Assign each finding an ID `R<r>.<k>` and a **fingerprint** = `dimension | file | the defect in ≤8 normalized words`. Fingerprints identify the same defect across rounds; judge "same defect" semantically, using the fingerprint as the key.

#### 1c. CI state
`gh pr checks <pr> --json name,bucket,link`. If any bucket is `pending`, wait (`wait_for` command, polling until no `pending` remains, ≤10 min per call, ≤30 min total). Failed required checks count as blocking items with source `ci:<name>`.

#### 1d. Decide
- `MERGE` and no failing checks → Phase 2.
- Otherwise there is a blocking set B_r (blocking findings + failing checks).
  - **Convergence check (r ≥ 2):** any fingerprint in B_r that was also in B_{r-1} after a fix round claimed to fix it → mark it `persisting`. Any fingerprint `persisting` for 2 consecutive rounds → **STALLED**. Do not retry it; stop the loop.
  - `r == max_rounds` → **STALLED**.
  - Cross-repo with no push rights → **READY-FOR-AUTHOR** reporting: post the findings (1e step 1) and stop.

#### 1e. Fix
1. Post the blocking set as ONE PR comment (`gh pr comment <pr> --body-file <tmp>`; never inline `--body`). The heading is `land round <r>: <n> blocking findings @ <reviewed_sha7>`, followed by one numbered item per finding: severity, `file:lines`, finding, suggestion. Failing checks are not reposted; /fix-pr reads CI itself. This comment is the handoff: /review never posts, and /fix-pr harvests issue-level comments.
2. Invoke `/fix-pr <pr>` directly, without `--re-review` (land's next round is the re-review).
3. Parse its result: pushed SHA(s), the per-item fix table, items `triaged-out` and `out-of-scope`. Map each back to `R<r>.<k>` and update the ledger fates.
4. /fix-pr BLOCKED → BLOCKED (carry its unblock condition and worktree path). /fix-pr reports "All spec items were triaged out" (its probes judged every blocker already addressed or invalid) → record each as `triaged-out` with the probe reasoning and continue to the next round; the re-review arbitrates, and if the reviewer re-raises the same fingerprint, the convergence rule in 1d turns the disagreement into STALLED rather than a loop.
5. `r += 1`, back to 1a.

### Phase 2: Merge gate (inline, every check must pass)

1. Re-fetch `gh pr view <pr> --json state,headRefOid,mergeable,mergeStateStatus`. MERGED by someone else → ALREADY-LANDED. CLOSED → BLOCKED.
2. `headRefOid == reviewed_sha`. If it differs, the head moved after review → go back to 1b with one extra round allowed (no fix needed, just re-review).
3. Checks green on `reviewed_sha` (re-run 1c).
4. `mergeable == MERGEABLE`. If `mergeStateStatus` is `BLOCKED` for branch protection (a required approving review), stop at **READY** naming the missing requirement. Never bypass protection, never `--admin`.
5. Authority: `--no-merge` → READY. Model-routed without `--auto-merge` → `ask_question` confirm ("Merge #N '<title>' at <sha7> via <strategy>?"). A decline or no handler → READY.
6. Merge:
   - squash: `gh pr merge <pr> --squash --match-head-commit <reviewed_sha> --subject "<PR title> (#<pr>)" --body-file <tmp>`. Build the body from the PR's commit messages with the Phase 0 operator rules applied (for example, rewritten co-author trailers).
   - merge/rebase: `gh pr merge <pr> --<strategy> --match-head-commit <reviewed_sha>`.
   - Add `--delete-branch` only when the head branch lives in this repo and the repo does not already auto-delete.
7. Verify: `gh pr view <pr> --json state,mergedAt,mergeCommit`. `state != MERGED` → BLOCKED with gh's error. Ledger: merge commit SHA.

### Phase 3: Follow-ups (skipped with `--no-issues`)

Candidates are every ledger finding whose fate is `waived-in-review` (severity ≥ low; nits only with `--nits`) or `out-of-scope`, plus human reviewer comments that explicitly say "follow-up", "later" or "separate PR". Then:
1. Group candidates that share a file and root cause into one issue. Do not file one issue per nit.
2. Dedupe each group with `gh issue list --state open --search "<2-4 distinctive terms> in:title,body" --limit 5`. A match → `gh issue comment <n> --body-file` linking this PR and finding, and set the fate to `existing #n`.
3. Otherwise `gh issue create --title "<conventional-prefix>: <defect>" --body-file <tmp>`. The body holds the source PR link, the merge commit, `file:lines` at the merge commit, the finding, the suggested fix, and severity. Add a `follow-up` label only if the repo already has one.
4. Ledger: the fate `filed #n` for each finding.

### Phase 4: Close and terminal state

1. Apply the remaining Phase 0 operator rules that fire after merge (for example, a required thank-you comment to a specific contributor, posted via `--body-file`).
2. Remove any worktree land created (`worktree remove`); /fix-pr cleans its own.
3. **Completeness table** from the ledger: every finding ID → fate. Any `unresolved` fate downgrades LANDED to BLOCKED.
4. Terminal verdict:

| Verdict | Meaning | Must cite |
|---|---|---|
| **LANDED** | Merged at the reviewed SHA, every finding has a fate | merge commit, rounds used, issues filed/linked, ledger path |
| **READY** | Final review MERGE + CI green, merge withheld (`--no-merge`, protection, declined confirm) | reviewed SHA, the exact merge command to run, missing requirement if any |
| **READY-FOR-AUTHOR** | Blocking findings posted; cannot push fixes to the head | comment URL |
| **STALLED** | Did not converge | persisting fingerprints with the attempted fixes, rounds used, recommended next skill (`/diagnose` on the persisting defect, or `/pivot-gate`) |
| **BLOCKED** | External or contract failure | exact unblock condition, worktree path if kept |
| **ALREADY-LANDED** | Merged before or during the run | merge commit, follow-ups still filed |

## Cost and pacing

A round is one /review (5 dimension agents plus verifiers) and one /fix-pr (probes plus one writer). Three rounds is the ceiling for a reason. Before each round, read `get_runtime_state` `usage[]`: if the entry for the provider this session runs on has `status` of `warn` or `over`, finish the current phase, write the ledger, and stop at READY or STALLED. Never start a review wave into a rate ceiling.

## Example

```
/land 2806
round 1: review DO NOT MERGE (1 high, 1 medium blocking; 2 low) @ 46ea77f → posted → /fix-pr pushed 9c1d2e0
round 2: review MERGE (0 blocking; 2 low, 1 nit) @ 9c1d2e0 → checks green
merge: squash --match-head-commit 9c1d2e0 → 5a7b... 
follow-ups: R1.3+R2.1 → #2961 (filed); R2.2 → existing #2930 (commented); R2.3 nit skipped
LANDED — 2 rounds, 1 issue filed, 1 linked, ledger .afk/land/pr-2806.md
```
