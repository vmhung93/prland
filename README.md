# prland

Review, approve, merge and clean up a GitHub pull request from the terminal,
with an AI review by [Claude Code](https://claude.com/claude-code).

```
prland service 123
```

1. **Review:** fetches the PR diff, leaves out lock and generated files, and
   asks Claude for a short colored report: verdict and risk level, highlight
   changes, risks (with the `path:line` and code that causes each), potential issues, suggestions, and what to check before merging.
2. **Merge readiness:** CI checks, merge conflicts, branch protection.
3. **Decide:** approve and merge, just approve, or leave it for a reviewer.
4. **Clean up:** offers to delete the merged branch (remote and local).

## Requirements

- `bash`, `git`, `awk`
- [`gh`](https://cli.github.com/), logged in: `gh auth login`
- [`claude`](https://claude.com/claude-code) (Claude Code CLI)

## Setup

```bash
git clone <this repo> ~/prland
cd ~/prland
chmod +x prland.sh
cp repos.conf.example repos.conf    # then edit it
ln -s ~/prland/prland.sh ~/bin/prland  # or any folder on your PATH
```

Optional tab completion (add to `~/.bashrc`):

```bash
eval "$(prland completion)"
```

## Configuration

`repos.conf` maps a short name to a local clone:

```ini
# name=path[|merge_method]     merge_method: merge (default) | squash | rebase
service=~/code/service
order-service=~/code/order-service|squash
admin=$HOME/admin-repo
```

- Paths may start with `~`, `$HOME` or `${HOME}`.
- SSH and HTTPS `origin` remotes both work. The remote is only used to find
  `owner/repo`; every GitHub action goes through `gh`.

The config is read from the first of these that exists:

1. `$PR_REVIEW_CONFIG`
2. `repos.conf` next to `prland.sh` (symlinks are followed)
3. `~/.config/prland/repos.conf`

## Usage

```bash
prland [--skip-review] <repo_name> <pr_number|pr_url>
prland [--skip-review] <pr_url>   # no config needed; skips local branch cleanup
prland list                       # show configured repos
prland --help
```

`--skip-review` skips the Claude review and goes straight to approve, then
merge (merge only for your own PR). It still shows merge readiness and asks
once before merging, and still offers branch cleanup. `claude` isn't needed
in this mode.

Reports are saved as `reports/<owner>_<repo>/<pr>-<sha>.md` next to the
script. If you run prland again on the same commit, it offers to reuse the
saved report instead of calling Claude again.

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `PR_REVIEW_CONFIG` | see above | Config file path |
| `PR_REVIEW_REPORT_DIR` | `reports/` next to the script | Where reports are saved |
| `PR_REVIEW_MERGE_METHOD` | `merge` | Default merge method |
| `PR_REVIEW_MODEL` | Claude Code default | Model passed to `claude --model` |
| `PR_REVIEW_MAX_DIFF_BYTES` | `400000` | Ask before reviewing larger diffs |
| `PR_REVIEW_PROTECTED` | `main master develop dev staging production release/* hotfix/* epic/*` | Branches never deleted |
| `PR_REVIEW_EXCLUDE` | lock files, `*.min.js`, `dist/`, `build/`, … | Regex of paths left out of the review |

## Safety

- Claude runs in an empty temp folder with all tools disabled. It only sees
  the diff and cannot touch your repo or GitHub; the script itself does every
  approve, merge and delete.
- The PR title, description and diff are marked as untrusted in the prompt.
- Merging and branch deletion always ask first. Branches that are protected on
  GitHub, match `PR_REVIEW_PROTECTED`, are on a fork, or are the base of other
  open PRs are never deleted.
- You cannot approve your own PR, so prland only offers to merge it.
