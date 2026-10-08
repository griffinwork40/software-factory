# tackle-issues

An [agent-afk](https://github.com/griffinwork40/agent-afk) skill that tackles N open GitHub issues in parallel using worktree-isolated subagents, producing one PR per issue.

PRs are proposals, not mutations. The safety net is [`/pr-triage`](https://github.com/griffinwork40/pr-triage) on the next pass.

## What it does

1. **Discover:** Fetches open issues, filters out already-claimed or PR-linked ones.
2. **Present:** Shows the hit list before dispatching.
3. **Parallel dispatch:** One `compose` node per issue, each in an isolated worktree. Capped at 5 concurrent.
4. **Report:** Summarizes PRs created, files changed, and any failures.

## Install

```bash
afk skill install griffinwork40/tackle-issues
```

Or clone directly into your skills directory:

```bash
git clone https://github.com/griffinwork40/tackle-issues.git ~/.afk/skills/tackle-issues
```

## Usage

```
/tackle-issues                          # Tackle up to 5 open issues
/tackle-issues 10                       # Tackle up to 10
/tackle-issues --issues 201,203,207     # Tackle specific issues
/tackle-issues --label bug              # Only issues labeled "bug"
/tackle-issues --skip "docs*"           # Skip documentation issues
/tackle-issues --repo owner/repo        # Target a different repo
/tackle-issues --draft                  # Create draft PRs
```

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `<N>` (positional) | 5 | How many issues to tackle |
| `--issues N,N,...` | auto-discover | Specific issue numbers |
| `--label <label>` | none | Filter by GitHub label |
| `--skip <glob>` | none | Skip issues matching pattern |
| `--repo owner/repo` | cwd remote | Target repository |
| `--draft` | off | Create draft PRs |

## Requirements

- [agent-afk](https://github.com/griffinwork40/agent-afk) installed and configured
- `gh` CLI authenticated (`gh auth status`)

## How it fits together

`/tackle-issues` is the issue-to-PR half of a two-skill loop:

- **`/tackle-issues`** burns down open issues into PRs (parallel, lightweight, no gates)
- **[/pr-triage](https://github.com/griffinwork40/pr-triage)** reviews, triages, merges, and fixes those PRs (sequential gates, human-approved)

## License

MIT
