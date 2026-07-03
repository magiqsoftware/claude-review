# Design: Switch Claude PR review to `workflow_dispatch`

**Date:** 2026-07-03
**Status:** Approved (pending spec review)
**Related:** Azure DevOps AB#33808 (original design)

## Problem

The reusable Claude PR review workflow is triggered by `issue_comment: [created]`
in each consuming repo's caller stub. A job-level `if:` guard then filters for
`@claude` (and excludes bot commenters). This guard works, but GitHub Actions
creates a **workflow run for every `issue_comment.created` event** and only
evaluates the job `if:` *after* the run exists. Comments that don't match produce
a 1-second **"Skipped"** run.

On a busy monorepo (e.g. `magiqsoftware/enterprise-module`) this means the Actions
tab fills with skipped runs — one per PR comment. This is inherent to the
`issue_comment` trigger: a job-level `if:` skips the *job*, not the *run*, and
`issue_comment` has no pre-run `paths`/`branches`-style filter.

## Goal

A review run is created **only when a user deliberately triggers one** — zero
skipped runs from ordinary PR comments. Preserve the existing options (model,
scope/`full`, `force` cost-cap bypass) and the `CLAUDE_REVIEW_USERS` allowlist.

## Chosen approach: `workflow_dispatch` with typed inputs

`workflow_dispatch` cannot be fired by a comment, so no run is ever created by PR
chatter. Reviews are triggered from the Actions "Run workflow" button or via
`gh workflow run`. Options become typed dispatch/`workflow_call` inputs, so the
comment-parsing step is removed entirely.

This is a **breaking change** to both the reusable workflow's `workflow_call`
signature and the caller stub's trigger. It ships as a new major release
**`v2.0.0`**. Existing `v1.x` consumers are unaffected until they migrate.

### Rejected alternatives

- **Keep `issue_comment`, hide the noise** (Actions Status filter) — cosmetic
  only; skipped runs still created. Rejected: user wants a robust fix.
- **Label trigger (`pull_request: [labeled]`)** — zero comment noise and stays on
  the PR, but options must be a fixed label vocabulary, not free-text. Rejected in
  favour of typed inputs which preserve full option flexibility.
- **Slash-command dispatch (`repository_dispatch`)** — keeps `/claude …` free-text
  but requires a first `issue_comment` workflow to parse comments, which
  *reintroduces* the per-comment skipped runs. Rejected: does not solve the problem.

## Detailed design

### Reusable workflow (`.github/workflows/claude-review.yml`)

**`on:` block** — replace the single `review_command` input with typed inputs plus
the existing `review_command` passthrough:

```yaml
on:
  workflow_call:
    inputs:
      pr_number:
        description: "PR number to review."
        type: number
        required: true
      model:
        description: "Model: sonnet | opus | haiku."
        type: string
        required: false
        default: sonnet
      scope:
        description: 'Review scope: "code errors" (default) or "all".'
        type: string
        required: false
        default: "code errors"
      force:
        description: "Bypass the per-PR review cap."
        type: boolean
        required: false
        default: false
      review_command:
        description: >-
          Slash command the CI step invokes. Defaults to the org toolkit review.
        type: string
        required: false
        default: "/pr-review-toolkit:review-pr"
```

Note the caller stub exposes `scope` as a `choice` of `"code errors"` / `full`
for UI friendliness, and maps `full` → `all` before passing it through (see
below). The reusable workflow receives the already-mapped scope string.

**Event-field replacements** — every reference is repointed:

| Today (comment context)                          | New                                   |
| ------------------------------------------------ | ------------------------------------- |
| `github.event.issue.number`                      | `inputs.pr_number`                    |
| `github.event.comment.body` (parsed)             | `inputs.model` / `inputs.scope` / `inputs.force` (typed) |
| `github.event.comment.user.login` (allowlist)    | `github.actor`                        |
| `github.event.comment.author_association` gate   | **removed** (redundant — see below)   |
| `github.event.comment.id` reactions              | **removed** (no comment to react to)  |

**Job-level `if:`** — reduce to the allowlist plus the PR guard is no longer
needed (dispatch always targets a PR by number):

```yaml
if: contains(fromJSON(vars.CLAUDE_REVIEW_USERS || '[]'), github.actor)
```

The `author_association` check is **dropped**: `workflow_dispatch` is only
available to users with repo write access, so OWNER/MEMBER/COLLABORATOR is
guaranteed for any allowed actor; the check is meaningless without a comment
event. The `CLAUDE_REVIEW_USERS` allowlist is retained as the finer, curated gate
on `github.actor`.

**"Parse review options" step** — **deleted**. Inputs arrive typed. Downstream
steps read `inputs.model` / `inputs.scope` / `inputs.force` directly (via `env:`
mappings, same shell-injection-safe pattern as today).

**Reaction steps** — the four `eyes` / `+1` / `confused` / `-1` reaction steps are
**deleted** (they targeted the trigger comment, which no longer exists).

**"Review started" acknowledgement** — a new first step posts an on-PR comment so
PR watchers see the review kicked off (replacing the lost `eyes` reaction):

```yaml
- name: Acknowledge trigger (review-started comment)
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    PR_NUMBER: ${{ inputs.pr_number }}
  run: |
    gh pr comment "$PR_NUMBER" --repo "${{ github.repository }}" \
      --body "🔍 Claude review started (triggered by @${{ github.actor }})…"
```

This comment carries **no** `REVIEW_MARKER`, so it does not inflate the cost-cap
count.

**`concurrency.group`** — `claude-review-${{ inputs.pr_number }}`.

**Unchanged logic** (repointed to `inputs.*` only): fork guard, cost cap check,
checkout, PR head checkout + base resolution, Node/npm cache, CLI + plugin
install, run Claude review, post review comment, report failure. The failure
comment (`⚠️ Claude review failed — see the Actions run log`) is retained; the
fork-guard skip condition on the failure step stays.

### Caller stub (`caller-stub/.github/workflows/claude-review.yml`)

```yaml
name: Claude PR Review

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: "PR number to review"
        required: true
        type: number
      model:
        description: "Model"
        type: choice
        options: [sonnet, opus, haiku]
        default: sonnet
      scope:
        description: "Review scope"
        type: choice
        options: ["code errors", "full"]
        default: "code errors"
      force:
        description: "Bypass the per-PR review cap"
        type: boolean
        default: false

permissions:
  pull-requests: write
  contents: read
  issues: write

jobs:
  review:
    uses: magiqsoftware/claude-review/.github/workflows/claude-review.yml@v2.0.0
    with:
      pr_number: ${{ inputs.pr_number }}
      model: ${{ inputs.model }}
      # Map the friendly "full" choice to the scope string the review command expects.
      scope: ${{ inputs.scope == 'full' && 'all' || 'code errors' }}
      force: ${{ inputs.force }}
    secrets: inherit
```

- The `if:` guard (comment-body / bot filter) and `types: [created]` are removed.
- Repos overriding the review command (e.g. `enterprise-cobol`'s `/cobol-review`)
  keep their `review_command:` line alongside the new `with:` inputs.
- The header comment is rewritten for the dispatch usage.

**Trigger UX:**
- Actions tab → "Claude PR Review" → **Run workflow** → enter PR number, pick
  model/scope/force → Run.
- CLI: `gh workflow run "Claude PR Review" -f pr_number=7764 -f model=opus -f scope=full`.

### Documentation

- **README** — rewrite "Usage (comment on a PR)" → "Usage (run the workflow)":
  replace the comment table with the dispatch inputs and a `gh workflow run`
  example. Update the `full` section to describe the `full`→`all` mapping via the
  `scope` choice input. Add a **v1 → v2 migration** note: replace the caller stub
  and bump the pin to `@v2.0.0`.
- **Caller-stub header comment** — update usage instructions to dispatch.
- Bump the `Releasing a new version` guidance if it names the current tag.

### Release

Cut **`v2.0.0`** (major — breaking `workflow_call` signature and trigger change),
following the repo's immutable-tag release process. Consuming repos migrate by
replacing their stub and bumping the pin; until then they stay on `v1.1.0`
unaffected.

## Trade-offs (accepted)

- **No `@claude` in-PR trigger.** Reviews are triggered from the Actions tab or
  `gh`. This is the deliberate cost of zero run-noise.
- **PR number entered manually** in the UI form (the `gh` one-liner is faster for
  CLI users).
- **Every consuming repo must migrate** (new stub + `@v2.0.0` pin): at minimum
  `enterprise-module` and `enterprise-cobol`.
- **No comment reactions**; feedback is the run status, the "review started"
  comment, the posted review, and the failure comment.

## Out of scope

- Auto-review on `pull_request` events (a separate feature, explicitly not wanted).
- Migrating consuming repos (tracked separately once `v2.0.0` is released).
- Any change to the review command / agent behaviour.
