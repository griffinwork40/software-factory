# software-factory

An [agent-afk](https://github.com/griffinwork40/agent-afk) plugin that turns a GitHub backlog into merged PRs.

It is three skills that form one loop:

| Skill | What it does | Gates |
|-------|--------------|-------|
| [`/tackle-issues`](skills/tackle-issues/SKILL.md) | Burns down N open issues in parallel, one worktree-isolated subagent and one PR per issue. | None. PRs are proposals, not mutations. |
| [`/pr-triage`](skills/pr-triage/SKILL.md) | Reviews open PRs in parallel, triages them into merge / fix / follow-up buckets, then merges and fixes. | A human gate on every irreversible action. |
| [`/land`](skills/land/SKILL.md) | Takes one PR all the way to merged: review, fix, re-review until nothing blocks (bounded), merge the exact reviewed SHA, file follow-up issues. | Typing `/land <PR>` is the merge approval; when a model routes it, the merge needs a confirm unless `--auto-merge` is passed. |

`/tackle-issues` produces PRs cheaply; `/pr-triage` is the safety net that decides what actually lands across the queue; `/land` drives a single PR to convergence when one needs more than one fix pass.

## Install

```bash
afk plugin install griffinwork40/software-factory
```

## Usage

```
/tackle-issues 3                         # Tackle 3 open issues, one PR each
/tackle-issues 5 --label bug             # Only issues labelled "bug"
/tackle-issues --issues 101,104 --draft  # Specific issues, open as draft PRs
/tackle-issues 5 --sort oldest            # Clear the backlog, oldest issues first
/tackle-issues 5 --screen                # Skip underspecified issues

/pr-triage                               # Review and triage all open PRs (up to 10)
/pr-triage --prs 101,102,105             # Triage specific PRs
/pr-triage --skip-fix                    # Triage and merge only
/pr-triage --sort oldest                 # Oldest PRs first
/pr-triage --release                     # Triage, merge, then draft a GitHub release

/land 101                                # Review, fix, re-review, merge, file follow-ups
/land 101 --no-merge                     # Converge, then stop at READY
/land 101 --max-rounds 2 --nits          # Tighter round cap; file nit follow-ups too
/land 101 --dry-run                      # First review only, print planned actions
/land 101 --repo ../other --auto-merge --no-issues  # Other checkout, no confirm, no issues
```

See each skill's `SKILL.md` for the full flag reference.

## Running it as a factory

Both skills run fine on demand. To run them unattended, schedule them with the agent-afk daemon: for example an hourly `/tackle-issues 1` job against a repo with well-scoped issues, plus a periodic `/pr-triage` sweep, with a human doing the final merge. Choose the gates you are comfortable with; `/pr-triage` keeps every merge human-approved unless you pass `--auto-merge`.

## Requirements

- [agent-afk](https://github.com/griffinwork40/agent-afk) installed and configured
- `gh` CLI authenticated (`gh auth status`)
- Sibling skills used by `/pr-triage` and `/land`: `/review`, `/fix-pr`, `/contract` (from [agent-workflow-amplifiers](https://github.com/griffinwork40/agent-workflow-amplifiers)). `/land` recommends `/diagnose` or `/pivot-gate` when it stalls.
- Optional: a `jev` MCP server (for example [`jev-mcp`](https://www.npmjs.com/package/jev-mcp)) with `TYPESAFE_API_KEY` set, for `/pr-triage`'s pre-filter wave. It is on by default for public repos and off for private ones, because PR diffs are sent to TypeSafe's API.

## History

The skills were previously published as separate repos, [`pr-triage`](https://github.com/griffinwork40/pr-triage) and [`tackle-issues`](https://github.com/griffinwork40/tackle-issues). Their full commit history is preserved here under `skills/`.

## License

MIT
