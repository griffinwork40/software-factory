# software-factory

## What This Is

An agent-afk plugin (`griffinwork40/software-factory`, MIT, v0.2.0) that turns a GitHub backlog into merged PRs. It holds no code: it is three prompt-only skills, each a single `SKILL.md`, that form one loop:

- `/tackle-issues`: burns down N open issues in parallel, one worktree-isolated subagent and one PR per issue. No gates; PRs are proposals.
- `/pr-triage`: reviews open PRs in parallel, buckets them (merge / fix / follow-up), then merges and fixes behind a human gate on every irreversible action.
- `/land`: takes one PR to merged through a bounded review, fix, re-review loop, merges the exact reviewed SHA, and files deduped follow-up issues.

`/tackle-issues` produces PRs cheaply; `/pr-triage` is the safety net that decides what lands across the queue; `/land` converges a single PR that needs more than one fix pass.

## Commands

There is no build, test, or lint step. The deliverable is markdown.

```bash
afk plugin install griffinwork40/software-factory   # install (registered subcommand, safe)
afk plugin update software-factory                  # pick up pushed changes
wc -l skills/*/SKILL.md                             # size check after edits
grep -n '^#' skills/*/SKILL.md                      # section outline
```

Verify a skill change by running it for real against a repo with open issues/PRs (for example `/pr-triage --prs <N> --skip-fix` as a low-risk smoke test). Never shell out to `afk <skill-name>`: unknown subcommands launch a nested agent session.

## Architecture

| Path | Purpose |
|------|---------|
| `.claude-plugin/plugin.json` | Plugin manifest (name, version, description, repo). Bump `version` on release. |
| `skills/tackle-issues/SKILL.md` | ~180 lines. Steps 1-4: discover issues, present hit list, parallel `agent` dispatch, report. |
| `skills/pr-triage/SKILL.md` | ~420 lines. Waves 0 (preflight), 0.5 (Jev pre-filter), 1 (parallel `/review`), 1.5 (triage gate), 2A/2B/2C (merge, fix, advisory issues), 3 (re-review). |
| `skills/land/SKILL.md` | ~165 lines. Single PR to merged. Phases 0 (preflight), 1 (review/fix round loop with convergence check), 2 (merge gate at reviewed SHA), 3 (deduped follow-up issues), 4 (close and verdict). `context: load`: must run top-level, never from a subagent. |
| `README.md` | User-facing overview, usage, requirements. Keep in sync with skill flags. |

Skill dependencies:
- `/pr-triage` and `/land` declare `skills: [review, fix-pr, contract]`; those live in other plugins (agent-workflow-amplifiers), not here.
- `/land` invokes `/review` and `/fix-pr` directly from the coordinator, so `/pr-triage` cannot delegate per-PR work to it from a subagent without a redesign.
- Wave 0.5 uses the optional `jev` MCP server (`TYPESAFE_API_KEY`). Default on for public repos, off for private ones because diffs leave the machine. It must stay fail-open.
- All skills require an authenticated `gh` CLI and fail closed without it.

History: both skills were imported with full history from the standalone repos `griffinwork40/pr-triage` and `griffinwork40/tackle-issues`. The plugin is installed from GitHub (`~/.afk/plugins/software-factory/`), so local edits take effect only after push plus `afk plugin update software-factory`. Do not recreate standalone copies under `~/.afk/skills/`; they would load each skill twice.

## Conventions

**SKILL.md frontmatter** (YAML, required keys): `name`, `description`, `argument-hint`, `category`, `when-to-use`, `flags`. Optional: `skills` (sibling skill deps), `failure_modes`, `max_tool_use_iterations`, `context` (execution context, e.g. `load`). Any new flag must appear in `argument-hint`, `flags`, the Arguments table, and README usage.

**SKILL.md body** follows a fixed shape: H1 title, `## Hard constraints` (where gates exist), `## Arguments` table (Flag | Default | Meaning), `## Execution` with numbered `### Wave N` or `### Step N` sections, fenced templates for operator-facing output, and an explicit terminal-state section.

**Dispatch rules encoded in the skills** (preserve when editing):
- Parallel fan-out uses one `agent` tool call per item, all issued in a single reply, with `isolation: "worktree"`. Not `compose` (commit `fd7cd1f`, `c9870de` moved away from it).
- Cap concurrency at 5; larger batches go in waves of 5.
- Merges are strictly sequential with a CI check between each. `--auto-merge` is the only bypass of the per-merge human gate.
- `/review` is read-only; only `/pr-triage` decides what gets posted to GitHub.
- Subagent commits and PR bodies go through temp files written with the `write_file` tool or a quoted heredoc redirected to a file (`cat > <file> <<'EOF'`), then referenced via `git commit -F <file>` or `gh pr create --body-file <file>`. Never inline with `-m "..."` or `--body "..."`. Never use stdin heredocs (`git commit -F - <<'EOF'`) because macOS /bin/sh (bash 3.2) mis-parses apostrophes inside them. Temp files go in the worktree or run dir, never `/tmp`.
- Subagents must not force-push, touch unrelated files, or combine issues.

**Writing style:** tables over prose for flags and buckets; every irreversible step names its gate. Existing section headings use `Wave N — Title`; keep that form for consistency, but avoid dashes in new prose.

**Commits:** conventional prefixes (`feat:`, `fix:`, `chore:`), messages via `git commit -F`. Agent identity is `AFK Agent <agent@agentafk.com>`.
