# Dependabot auto-merge

`.github/workflows/dependabot-auto.yml` decides what happens to a Dependabot pull request once this repository's
`CI` workflow has finished on it. It exists because the repository has no required status checks, so
`gh pr merge --auto` would merge immediately with no gate at all.

## Trigger and gate

The workflow runs on `workflow_run` of `CI` (completed) for pull requests opened by `dependabot[bot]`. It acts only
when the PR is still open and its head is the SHA that CI tested. The update type comes from the
`update-type: version-update:semver-*` line Dependabot writes into its commit message; a PR without that line is left
to a human.

| Update type | CI on the PR head | Result |
| --- | --- | --- |
| patch or minor | green | squash-merged with `--match-head-commit <tested head>` |
| patch or minor | failed | left open |
| major | green | left open for a human |
| major | failed, `security` label or GHSA/CVE text in the body | left open (a closed PR would mark the vulnerable range ignored) |
| major | failed, default-branch CI at its current tip green | closed with a comment; Dependabot treats a closed PR as ignoring that version |
| major | failed, default-branch CI not green | left open |

Before a patch or minor merge the job requires every check run on the head (except the retired `Workers Builds`
check) to be `success`, `neutral` or `skipped`, and every legacy commit status to be `success`. Pending checks and a
pending default-branch run are polled (30 s, up to 40 tries; the job times out at 30 minutes).

## Concurrency and a stale base

Decisions are serialized per repository (`concurrency: dependabot-auto`). Under that lock the job requires the head to
contain the current default-branch tip; if it does not, the branch is updated and the new CI run decides.
`--match-head-commit` pins only the PR head, never the base. After a merge the job updates the other open Dependabot
branches so their CI re-runs on the new base. GitHub keeps one pending run per concurrency group, so a dropped
decision is recovered by that post-merge update.

## Credentials

Without an App, the job uses `GITHUB_TOKEN`. Events caused by `GITHUB_TOKEN` do not start `push` or `pull_request`
workflows, so a merge made that way does not run deploy-on-push workflows and a branch update made that way does not
re-run CI. Repositories with a deploy-on-push workflow therefore need the App path:

- Dependabot secrets `DEPENDABOT_AUTO_APP_ID` and `DEPENDABOT_AUTO_APP_KEY` (Settings, Secrets, Dependabot).
- The workflow mints a token limited to this repository with `contents` and `pull-requests` write and `actions`,
  `checks` and `statuses` read, using a pinned `actions/create-github-app-token`.

Dependabot secrets are readable by workflows that Dependabot triggers, including CI install steps. Keep the App
narrowly permissioned and installed only on repositories that use this workflow.

## Operating it

- To retry a decision, re-run the `CI` workflow on the PR (or comment `@dependabot rebase`).
- To stop auto-merge for one PR, add a non-Dependabot commit to its branch or close the PR.
- To disable the lane, delete the workflow file.
