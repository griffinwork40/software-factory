# pr-triage

An [agent-afk](https://github.com/griffinwork40/agent-afk) skill that batch-reviews open PRs in parallel, triages them into merge/fix/follow-up buckets, then executes each bucket with human gates on every irreversible operation.

## What it does

1. **Wave 0 -- Preflight:** Fetches open PRs and builds a manifest table.
2. **Wave 0.5 -- Jev pre-filter (optional):** If the [Jev](https://typesafe.ai) MCP server is connected, Jev reads each PR's diff server-side (the diffs never enter the agent's context) and answers three questions: is the change trivial, is it ready for review, and how risky is the code it touches. Trivial, low-risk PRs get a light review on a cheaper model; drafts and unfinished PRs are deferred; the rest get the normal full review, riskiest first. Jev never decides whether a PR merges, and any Jev error falls back to a full review. On by default for public repos, off for private ones (diffs would be sent to TypeSafe's API).
3. **Wave 1 -- Parallel Review:** Dispatches `/review` per PR as parallel agent calls (capped at 5 concurrent).
4. **Wave 1.5 -- Triage Gate:** Classifies each PR into GREEN (merge), BLOCKED (fixable), or SKIP (including deferred PRs, which you can pull back in with `include <N>`).
5. **Wave 2 -- Execute Buckets:** Sequential merges with CI checks, parallel `/fix-pr` dispatches for blocked PRs, and optional advisory issues for non-blocking findings on merged PRs. Every irreversible action is human-gated.
6. **Wave 3 -- Re-review:** Optionally re-reviews fixed PRs and offers to merge.

## Install

```bash
afk skill install griffinwork40/pr-triage
```

Or clone directly into your skills directory:

```bash
git clone https://github.com/griffinwork40/pr-triage.git ~/.afk/skills/pr-triage
```

## Usage

```
/pr-triage                              # Triage all open PRs (up to 10)
/pr-triage --prs 101,102,105            # Triage specific PRs
/pr-triage --limit 20                   # Raise the discovery limit
/pr-triage --repo owner/repo            # Target a different repo
/pr-triage --auto-merge                 # Skip human gate for green PR merges
/pr-triage --skip-fix                   # Triage and merge only, skip fix phase
```

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--prs N,N,...` | all open | Specific PR numbers to triage |
| `--limit N` | 10 | Max PRs to fetch in discovery |
| `--repo owner/repo` | cwd remote | Target repository |
| `--auto-merge` | off | Batch-merge green PRs (still sequential + CI-gated) |
| `--skip-fix` | off | Skip the fix phase |
| `--prefilter` | on for public repos | Force the Jev pre-filter on, including for private repos |
| `--no-prefilter` | off | Skip the Jev pre-filter; every PR gets a full review |

## Requirements

- [agent-afk](https://github.com/griffinwork40/agent-afk) installed and configured
- `gh` CLI authenticated (`gh auth status`)
- Sibling skills: `/review`, `/fix-pr`
- Optional: a `jev` MCP server (for example [`jev-mcp`](https://www.npmjs.com/package/jev-mcp), pinned to a version) with `TYPESAFE_API_KEY` set, for the Wave 0.5 pre-filter

## How it fits together

`/pr-triage` is the review-and-merge half of a two-skill loop:

- **[/tackle-issues](https://github.com/griffinwork40/tackle-issues)** burns down open issues into PRs (parallel, lightweight, no gates)
- **`/pr-triage`** reviews, triages, merges, and fixes those PRs (sequential gates, human-approved)

## License

MIT
