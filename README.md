# metplus-action-claude-code-review

METplus Claude Code Review Action

A GitHub Composite Action used to review pull requests in the METplus
component repositories with [Claude Code](https://claude.com/claude-code).
Keeping the review prompt and its settings in one place means every
repository gets the same review, and a change to it is made once.

## Version History

### v1
* **2026-11-17**
* Initial version, based on the MET `claude-code-review.yml` workflow
  (dtcenter/MET#3553, dtcenter/MET#3567)

---

## Description

This composite action:
* Checks that the inputs are of the expected form before using them.
* Checks out the repository with full history: the pull request merge commit
  for a `pull_request` event, or the pull request head for any other event,
  such as `workflow_dispatch`.
* Runs the Claude Code `/code-review` command against the pull request, which
  posts inline comments on the diff followed by one top-level summary comment.

The summary comment describes the change, lists the findings, judges the pull
request against the conventions recorded in the repository's top-level
`AGENTS.md` and the guides it points to, and lists questions for the author.
Repository-specific conventions belong in `AGENTS.md`, not in this action, so
every repository can call it the same way.

---

## Inputs

| Input | Description | Required | Default |
| :--- | :--- | :---: | :--- |
| **claude-code-oauth-token** | Claude Code OAuth token, normally `secrets.CLAUDE_CODE_OAUTH_TOKEN`. | Yes | |
| **pr-number** | Number of the pull request to review. | Yes | |
| **effort** | Review effort level: `low`, `medium`, `high`, `xhigh`, or `max`. `low` reports only a few high-confidence findings; `high` reports more, including some the reviewer is less certain about. | No | `high` |
| **max-turns** | Maximum number of turns Claude may take. | No | `80` |

---

## Usage

The calling workflow decides when a review runs and grants the permissions it
needs. The triggers, the job condition, the permissions, the concurrency group,
and the timeout cannot be set inside a composite action.

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [opened, ready_for_review, labeled]
    branches-ignore:
      - '*-ref'
  workflow_dispatch:
    inputs:
      pr_number:
        description: 'Pull request number to review'
        required: true
        type: string

concurrency:
  group: claude-review-${{ github.event.pull_request.number || inputs.pr_number }}
  cancel-in-progress: true

jobs:
  claude-review:
    if: |
      github.event_name == 'workflow_dispatch' ||
      (github.event.pull_request.draft == false &&
       (github.event.action != 'labeled' ||
        github.event.label.name == 'claude-review'))
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      pull-requests: write
      issues: read
      id-token: write
    steps:
      - uses: dtcenter/metplus-action-claude-code-review@v1
        with:
          claude-code-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          pr-number: ${{ github.event.pull_request.number || inputs.pr_number }}
```

---

## Security

The calling workflow passes the OAuth token to this action as an input. The
action runs in the caller's job, so it can reach nothing the job could not
reach already, and GitHub masks the token in the logs. Anyone who can push to
this repository or move its tags can change the code that receives the token,
so keep write access to this repository limited to the maintainers of the
METplus component repositories.

The token is tied to the Claude subscription of the person who generated it,
and usage from every repository that calls this action counts against that
person's account.

GitHub withholds repository secrets from `pull_request` runs for pull
requests opened from a fork of a public repository, so no review runs for
those.
