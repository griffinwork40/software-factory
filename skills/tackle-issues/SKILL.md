---
name: tackle-issues
description: "Tackle N open GitHub issues in parallel using worktree-isolated subagents, producing one PR per issue. Lightweight wrapper — PRs are proposals, not mutations; the safety net is /pr-triage on the next pass."
argument-hint: "<N> [--label <label>] [--skip <glob>] [--issues <N,N,...>] [--repo <owner/repo>] [--draft]"
category: "Build & ship"
when-to-use: "When you have open issues to burn down in parallel. Triggers on 'tackle issues', 'work on open issues', 'burn down issues', 'make PRs for issues'."
flags: [--label, --skip, --issues, --repo, --draft]
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

## Execution

### Step 1 — Discover issues

If `--issues` provided, fetch those directly. Otherwise:

```bash
gh issue list --state open --limit <N * 2> --json number,title,body,labels,assignees
```

Filter out issues that:
- Already have an open PR linked (check: `gh pr list --search "issue:<number>"` or scan for branch naming conventions)
- Are assigned to someone else
- Match `--skip` pattern
- Don't match `--label` filter (if set)

Take the first `<N>` issues that pass filters.

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

### Step 3 — Parallel dispatch

Dispatch one `agent` tool call per issue. Issue all calls for the current wave in a **single reply** so they run in parallel.

- `agent_type: "general-purpose"`
- `isolation: "worktree"` — each agent gets its own worktree; dirty/ahead worktrees are preserved and locked by the runtime; clean ones are torn down automatically
- `model: "claude-sonnet-4-6"`
- Cap at **5 concurrent**. If there are more than 5 issues, batch them into waves of 5: dispatch wave 1 (5 calls in one reply), wait for all 5 to complete, then dispatch wave 2, and so on.

Each agent's prompt:

```
You are fixing GitHub issue #<number> in <repo>.

**Issue title:** <title>
**Issue body:**
<body>

Instructions:
1. Read the issue carefully. Understand the specific ask.
2. Find the relevant file(s) and make the fix.
3. Run the project's test command if a test file is relevant to your change.
4. Run `pnpm lint` (or the project's lint command) to verify your change compiles.
5. Commit with message: `fix(#<number>): <short description>`
   Use `git commit -F <tmpfile>` — never inline the message with -m.
6. Create a PR:
   - `gh pr create --title "fix(#<number>): <short description>" --body-file <tmpfile>`
   - PR body should reference `Closes #<number>` and briefly describe what changed.
   - Add `--draft` flag if instructed.
7. Report what you did: PR URL, files changed, tests run.

Do NOT:
- Touch files unrelated to the issue
- Make architectural changes or large refactors
- Combine multiple issues into one PR
- Force push or modify other branches
```

### Step 4 — Report results

After all agent calls complete (all waves if batched), collect each agent's final message and summarize:

```
## Results

| # | Issue | PR | Status | Summary |
|---|-------|----|--------|---------|
| 201 | Add null check in auth.ts:47 | #212 | ✅ | Added guard + test |
| 203 | Remove unused import | #213 | ✅ | Removed 3 unused imports |
| 207 | Add parser test | — | ❌ | Couldn't reproduce edge case |

<N>/<M> issues tackled. Run `/pr-triage` to review and merge.
```

Worktree lifecycle is managed automatically by the `agent` tool when `isolation: "worktree"` is used — no manual cleanup needed. Dirty or ahead worktrees are preserved and locked so the operator can inspect; clean worktrees are torn down by the runtime.

### That's it

No merge gates, no human checkpoints on PR creation — PRs are proposals.
The safety net is `/pr-triage` on the next pass, which reviews with full gates.
