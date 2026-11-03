# Ruleset workflows

Workflows that `actions-fyi` organization rulesets run in other repositories, through the rule
**Require workflows to pass before merging**. They live here once, instead of being copied into each
repository.

| Workflow | File | What it does |
|---|---|---|
| Review | [`claude-review.yml`](.github/workflows/claude-review.yml) | Claude reviews each pull request and posts inline comments and a summary. Advisory: it never blocks merging. |

## Setting up the ruleset

1. **Organization settings → Repository → Rulesets → New ruleset → New branch ruleset.**
2. Target the repositories that should run the workflow, and only branches that are changed through
   pull requests (e.g. the default branch). The rule blocks direct pushes to the targeted branches.
3. Add **Require workflows to pass before merging**, with repository `actions-fyi/ruleset-workflows`,
   the workflow path, and a pinned ref: a commit SHA or tag. A change here then only reaches the
   targets once the ruleset is updated.

A ruleset doesn't start on pull requests that are already open. They pick it up with their next push.

## Behaviour of ruleset workflows

- Only `pull_request`, `pull_request_target` and `merge_group` trigger them, always with the default
  activity types: `opened`, `synchronize` and `reopened` for pull requests. Filters such as `types`,
  `branches` and `paths` are ignored.
- They run in the context of the target repository: its `GITHUB_TOKEN`, its secrets and variables.
- They don't run in this repository: each job checks `github.repository`, so it isn't run a second
  time as a regular workflow.

## Review

Needs, shared with every target repository:

| Name | Kind | Value |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | organization secret | output of `claude setup-token` |
| `CLAUDE_CODE_OAUTH_TOKEN_EXPIRES_AT` | organization variable | the token's expiry date, e.g. `2027-11-03` |

- **Skipped** for drafts, bot pull requests, Renovate and Dependabot branches, and forks (which get
  no secrets). A draft marked as ready is reviewed on its next push, because rulesets don't run on
  `ready_for_review`.
- **Scope:** a push that only adds commits on top of the last reviewed commit gets an incremental
  Sonnet review of those commits. Anything else gets a full review: opening or reopening the pull
  request, no earlier review, rewritten history, or a merge commit among the new commits.
- **Token check:** fails the run if the secret isn't available, warns 30 days before the expiry
  date, and fails once it has passed.
- **Identity:** the review uses the workflow's `GITHUB_TOKEN`, so comments come from
  `github-actions[bot]`. The Claude GitHub App's token exchange requires the workflow file on the
  target repository's default branch, which a ruleset workflow never is.
- **Advisory:** the job has `continue-on-error`, so a failed run (expired token, unshared secret,
  outage) shows as a failed job but doesn't block merging.
