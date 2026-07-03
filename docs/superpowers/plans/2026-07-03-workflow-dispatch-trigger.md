# workflow_dispatch Review Trigger Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the `issue_comment` trigger with `workflow_dispatch` so review runs are only created on deliberate action (no skipped-run noise), with model/scope/force as typed inputs.

**Architecture:** The reusable workflow (`.github/workflows/claude-review.yml`) gains typed `workflow_call` inputs and drops all comment-event references. The caller stub (`caller-stub/.github/workflows/claude-review.yml`) swaps `issue_comment` for `workflow_dispatch` and passes inputs through. A single PR comment is posted then edited in place across the run lifecycle. Ships as breaking release `v2.0.0`.

**Tech Stack:** GitHub Actions YAML, `gh` CLI, bash. Verification via `python3 -c 'import yaml; yaml.safe_load(...)'` (PyYAML confirmed available; Python 3.8.10) — no test framework in this repo.

## Global Constraints

- Reusable workflow file: `.github/workflows/claude-review.yml`. Caller stub template: `caller-stub/.github/workflows/claude-review.yml`. Both live in THIS repo and change together.
- Comment bodies are always passed to shell via `env:`, never interpolated into the script (shell-injection safety). Preserve this pattern for any new `gh` calls.
- `REVIEW_MARKER` = `<!-- claude-review:complete -->`. It is added **only** to a completed review body — never to a "started" or "capped" comment. The cost cap counts comments containing it.
- `CLAUDE_CODE_VERSION` pin (`2.1.185`) is retained unchanged.
- New release tag is `v2.0.0` (breaking). The caller stub's `uses:` pin becomes `@v2.0.0`.
- Scope mapping: caller exposes `choice` of `"code errors"` / `"full"`; caller maps `full` → `all` before passing `scope` to the reusable workflow. The reusable workflow receives the already-mapped string (`"code errors"` or `"all"`).
- Cost cap threshold stays at 3; `force` bypasses it. `force` is now a boolean input, not a comment keyword.
- Every consuming-repo migration is OUT OF SCOPE (tracked separately once v2.0.0 is tagged).

---

### Task 1: Rework the reusable workflow to `workflow_call` typed inputs

**Files:**
- Modify: `.github/workflows/claude-review.yml`

**Interfaces:**
- Consumes: nothing (entry point).
- Produces: `workflow_call` signature with inputs `pr_number` (number, required), `model` (string, default `sonnet`), `scope` (string, default `"code errors"`), `force` (boolean, default `false`), `review_command` (string, default `"/pr-review-toolkit:review-pr"`). Job id `review`. Step ids: `forkguard`, `cap`, `startcomment`, `pr`, `review`.

This task is large but atomic — the workflow is one file and its parts are interdependent (removing the parser requires repointing every downstream step in the same edit). It ends with a YAML-valid, self-consistent workflow.

- [ ] **Step 1: Replace the `on:` block**

In `.github/workflows/claude-review.yml`, replace the entire `on:` block (currently lines ~27-39) with:

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
        description: 'Review scope passed to the review command: "code errors" or "all".'
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
          Slash command the CI step invokes. Defaults to the org toolkit
          review. A consuming repo may point this at its own
          .claude/commands/<name>.md to run a repo-tuned review instead.
        type: string
        required: false
        default: "/pr-review-toolkit:review-pr"
```

- [ ] **Step 2: Update the job-level `if:` guard**

Replace the current `if:` (the four-line comment-context guard) with the allowlist-only gate. Keep the explanatory comment above it accurate:

```yaml
    # Gate on the curated allowlist. workflow_dispatch is already restricted to
    # users with repo write access, so author_association is guaranteed and no
    # longer checked; CLAUDE_REVIEW_USERS is the finer, curated gate on the actor.
    if: contains(fromJSON(vars.CLAUDE_REVIEW_USERS || '[]'), github.actor)
```

- [ ] **Step 3: Update `concurrency.group`**

Change `group: claude-review-${{ github.event.issue.number }}` to:

```yaml
  group: claude-review-${{ inputs.pr_number }}
```

- [ ] **Step 4: Delete the four reaction steps**

Delete these steps entirely (they targeted the trigger comment, which no longer exists):
- `Acknowledge trigger (eyes reaction)`
- `Acknowledge success (+1 reaction)`
- `Acknowledge skipped (confused reaction)`
- `Acknowledge failure (-1 reaction)`

- [ ] **Step 5: Delete the "Parse review options" step**

Delete the whole `Parse review options` step (id `args`) — inputs now arrive typed. All later `steps.args.outputs.*` references are repointed in the steps below.

- [ ] **Step 6: Reorder — fork guard first, then cost cap, then started-comment**

Keep `Refuse cross-repository (fork) PRs` (id `forkguard`) as the first step but repoint its PR number env to `inputs.pr_number`:

```yaml
      - name: Refuse cross-repository (fork) PRs
        id: forkguard
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ inputs.pr_number }}
        run: |
          CROSS=$(gh pr view "$PR_NUMBER" \
            --repo "${{ github.repository }}" \
            --json isCrossRepository -q .isCrossRepository)
          if [ "$CROSS" = "true" ]; then
            gh pr comment "$PR_NUMBER" --repo "${{ github.repository }}" \
              --body "🚫 Claude review is disabled for cross-repository (fork) PRs for security reasons (the workflow runs with repository secrets). Re-run from a branch in this repository."
            echo "Refusing to review a cross-repository (fork) PR with secrets present." >&2
            exit 1
          fi
```

- [ ] **Step 7: Repoint the cost-cap step and its `force` source**

The `Cost cap check` step (id `cap`) keeps its counting logic. Repoint PR number to `inputs.pr_number`, read `force` from `inputs.force`, and reword the capped message to point at the `force` input:

```yaml
      - name: Cost cap check
        id: cap
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ inputs.pr_number }}
          FORCE: ${{ inputs.force }}
        run: |
          COUNT=$(gh pr view "$PR_NUMBER" \
            --repo "${{ github.repository }}" \
            --json comments \
            -q "[.comments[] | select(.body | contains(\"$REVIEW_MARKER\"))] | length")
          echo "This PR has $COUNT completed Claude review(s); force=$FORCE."

          if [ "$COUNT" -ge 3 ] && [ "$FORCE" != "true" ]; then
            echo "capped=true" >> "$GITHUB_OUTPUT"
            # Deliberately NO marker in this comment, so it doesn't inflate the count.
            gh pr comment "$PR_NUMBER" --repo "${{ github.repository }}" --body \
          "⏭️ Skipping this Claude review: this PR has already had **$COUNT** reviews, and reviews are capped at 3 to limit cost. Re-run the workflow with the **force** input enabled (Actions → Run workflow → tick *force*, or \`gh workflow run … -f force=true\`) to run one anyway."
            echo "Cost cap reached — declining this review." >&2
          else
            echo "capped=false" >> "$GITHUB_OUTPUT"
          fi
```

- [ ] **Step 8: Add the "Post review-started comment" step**

Immediately after the cost-cap step, add a step that posts the single status comment and captures its id. `gh pr comment` prints the created comment URL whose trailing `-<id>` is the comment id:

```yaml
      - name: Post review-started comment
        id: startcomment
        if: success() && steps.cap.outputs.capped != 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ inputs.pr_number }}
          ACTOR: ${{ github.actor }}
        run: |
          URL=$(gh pr comment "$PR_NUMBER" --repo "${{ github.repository }}" \
            --body "🔍 Claude review started (triggered by @${ACTOR})…")
          echo "id=${URL##*-}" >> "$GITHUB_OUTPUT"
          echo "Started comment id: ${URL##*-}"
```

- [ ] **Step 9: Repoint checkout / PR-head / setup steps**

For each of these steps, replace `${{ github.event.issue.number }}` with `${{ inputs.pr_number }}` and leave everything else unchanged:
- `Check out PR head and resolve base branch` (id `pr`): the two `github.event.issue.number` uses in `gh pr checkout` and `gh pr view`.

The `Checkout`, `Setup Node`, `Cache npm downloads`, `Install Claude Code CLI`, `Install pr-review-toolkit plugin` steps have no comment references — leave their bodies unchanged (their `if: success() && steps.cap.outputs.capped != 'true'` guards stay).

- [ ] **Step 10: Repoint the "Run Claude review" step to typed inputs**

In the `Run Claude review` step (id `review`), repoint the env block:

```yaml
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          PR_NUMBER: ${{ inputs.pr_number }}
          BASE_REF: ${{ steps.pr.outputs.base }}
          REVIEW_MODEL: ${{ inputs.model }}
          REVIEW_SCOPE: ${{ inputs.scope }}
```

The `PROMPT` body and `claude -p` invocation are unchanged (it already reads `${REVIEW_SCOPE}`, `${PR_NUMBER}`, `${BASE_REF}`, `${REVIEW_MODEL}` and `${{ inputs.review_command }}`).

- [ ] **Step 11: Convert "Post review comment" into an in-place edit**

Replace the `Post review comment` step so it edits the started-comment (via the Issues comment PATCH endpoint) instead of posting a new one. Keep the empty-output guard, the marker, and the local-run footer:

```yaml
      - name: Update status comment with the review
        if: success() && steps.cap.outputs.capped != 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          COMMENT_ID: ${{ steps.startcomment.outputs.id }}
          REVIEW_MODEL: ${{ inputs.model }}
          REVIEW_SCOPE: ${{ inputs.scope }}
        run: |
          if [ ! -s review.md ]; then
            echo "Review output was empty" >&2
            exit 1
          fi

          # Append the hidden completion marker (counted by the cost cap) and a
          # collapsed how-to for running the same review locally.
          {
            printf '\n\n---\n'
            printf '%s\n' "$REVIEW_MARKER"
            printf '_Reviewed by Claude (`%s`, scope: `%s`)._\n\n' "$REVIEW_MODEL" "$REVIEW_SCOPE"
            printf '<details><summary>Run a Claude review locally</summary>\n\n'
            printf '```sh\n'
            printf 'claude plugin marketplace add anthropics/claude-plugins-official\n'
            printf 'claude plugin install pr-review-toolkit@claude-plugins-official\n'
            printf 'claude "/pr-review-toolkit:review-pr"\n'
            printf '```\n\n'
            printf '</details>\n'
          } >> review.md

          gh api --method PATCH \
            -H "Accept: application/vnd.github+json" \
            "/repos/${{ github.repository }}/issues/comments/${COMMENT_ID}" \
            -F body=@review.md
```

- [ ] **Step 12: Convert the failure step to edit-or-post**

Replace `Report failure` so it edits the started-comment when one exists, else posts a fresh comment (covers failures before the started-comment, e.g. fork guard). Keep the fork-guard skip condition:

```yaml
      - name: Report failure
        # Skip when the fork guard already posted its own explanatory comment.
        if: failure() && steps.forkguard.outcome != 'failure'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ inputs.pr_number }}
          COMMENT_ID: ${{ steps.startcomment.outputs.id }}
        run: |
          RUN_URL="${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          BODY="⚠️ Claude review failed — see the Actions run log: ${RUN_URL}"
          if [ -n "$COMMENT_ID" ]; then
            gh api --method PATCH \
              -H "Accept: application/vnd.github+json" \
              "/repos/${{ github.repository }}/issues/comments/${COMMENT_ID}" \
              -f body="$BODY"
          else
            gh pr comment "$PR_NUMBER" --repo "${{ github.repository }}" --body "$BODY"
          fi
```

- [ ] **Step 13: Update the top-of-file header comment**

Update the leading comment block (lines ~1-24) so the trigger grammar and example reflect dispatch, not `@claude`. Replace the "Trigger grammar (in the caller's PR comment)…" paragraph with a description of the `workflow_dispatch` inputs (`pr_number`, `model`, `scope`, `force`) and note the caller owns the `workflow_dispatch` trigger. Keep the allowlist and cost-cap notes.

- [ ] **Step 14: Validate YAML**

Run:

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/claude-review.yml')); print('valid yaml')"
```

Expected: `valid yaml`

- [ ] **Step 15: Grep for stale comment-event references**

Run:

```bash
grep -nE 'github\.event\.(comment|issue)|steps\.args|reaction' .github/workflows/claude-review.yml && echo "FOUND STALE REFS" || echo "clean"
```

Expected: `clean` (no matches). If any line prints, it is a missed repoint — fix it and re-run.

- [ ] **Step 16: Commit**

```bash
git add .github/workflows/claude-review.yml
git commit -m "feat!: trigger reusable review via workflow_dispatch typed inputs

BREAKING CHANGE: workflow_call signature now takes pr_number/model/scope/force
instead of reading the issue_comment event. Removes comment parsing, reactions,
and the author_association gate; posts one status comment edited in place.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Rewrite the caller stub for `workflow_dispatch`

**Files:**
- Modify: `caller-stub/.github/workflows/claude-review.yml`

**Interfaces:**
- Consumes: the reusable workflow's `workflow_call` inputs from Task 1 (`pr_number`, `model`, `scope`, `force`, `review_command`) at pin `@v2.0.0`.
- Produces: the copy-paste template consumers adopt.

- [ ] **Step 1: Replace the whole caller stub**

Replace the entire contents of `caller-stub/.github/workflows/claude-review.yml` with:

```yaml
# Claude PR Review — caller stub (copy this into a consuming repo).
#
# The review logic lives in the org-wide reusable workflow at
# magiqsoftware/claude-review. This thin file owns the workflow_dispatch trigger
# and delegates everything else. To adopt Claude review in another repo, copy
# this file into <repo>/.github/workflows/ and ensure ANTHROPIC_API_KEY (secret)
# and CLAUDE_REVIEW_USERS (variable) are visible to the repo (org- or repo-level).
#
# Trigger a review: Actions tab → "Claude PR Review" → Run workflow → enter the
# PR number and pick options. Or from the CLI:
#   gh workflow run "Claude PR Review" -f pr_number=1234 -f model=opus -f scope=full
#
# Options:
#   pr_number   PR to review (required)
#   model       opus | sonnet | haiku   (default: sonnet)
#   scope       "code errors" | full    (default: code errors)
#   force       bypass the per-PR review cap (default: false)
#
# Updating: to pick up upstream changes, bump the @vX.Y.Z pin below to a newer
# claude-review release tag in a reviewed PR (never a moving alias like @v2).
# See the claude-review README for the release/versioning process.
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

# The effective GITHUB_TOKEN permissions are the intersection of these and the
# reusable workflow's. Declare them here so the token is write-capable.
permissions:
  pull-requests: write
  contents: read
  issues: write

jobs:
  review:
    # Pinned to an immutable release tag, not a moving major alias: this job runs
    # with this repo's secrets, so a change to the central workflow must be adopted
    # via a reviewed version bump here rather than taking effect silently.
    uses: magiqsoftware/claude-review/.github/workflows/claude-review.yml@v2.0.0
    with:
      pr_number: ${{ inputs.pr_number }}
      model: ${{ inputs.model }}
      # Map the friendly "full" choice to the scope string the review command expects.
      scope: ${{ inputs.scope == 'full' && 'all' || 'code errors' }}
      force: ${{ inputs.force }}
    # A repo can run its own review command instead of the default toolkit review
    # by adding, e.g.:
    #     review_command: "/my-review"
    secrets: inherit
```

- [ ] **Step 2: Validate YAML**

Run:

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('caller-stub/.github/workflows/claude-review.yml')); print('valid yaml')"
```

Expected: `valid yaml`

- [ ] **Step 3: Confirm the pin and trigger are correct**

Run:

```bash
grep -nE 'workflow_dispatch|@v2\.0\.0|issue_comment|@claude' caller-stub/.github/workflows/claude-review.yml
```

Expected: matches for `workflow_dispatch` and `@v2.0.0`; **no** matches for `issue_comment` or `@claude`.

- [ ] **Step 4: Commit**

```bash
git add caller-stub/.github/workflows/claude-review.yml
git commit -m "feat!: caller stub triggers via workflow_dispatch (pin @v2.0.0)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Update README for the dispatch trigger and v1→v2 migration

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: the input names/behaviour from Tasks 1-2.
- Produces: user-facing docs.

- [ ] **Step 1: Update the intro paragraph**

In `README.md`, replace the opening description that says "Comment `@claude` on a pull request…" with a dispatch description, e.g.:

```markdown
Org-wide **reusable** GitHub Actions workflow for on-demand Claude PR reviews.
Run the "Claude PR Review" workflow on a pull request (Actions tab or `gh
workflow run`) and a multi-agent code review is posted back as a PR comment. The
logic lives here once; consuming repos add a thin `workflow_dispatch` stub.
```

- [ ] **Step 2: Replace the "Usage (comment on a PR)" section**

Replace the section titled `## Usage (comment on a PR)` and its comment table with:

````markdown
## Usage (run the workflow)

Trigger from the Actions tab → **Claude PR Review** → **Run workflow**, or via the
CLI:

```sh
gh workflow run "Claude PR Review" -f pr_number=1234 -f model=opus -f scope=full
```

| Input | Values | Effect |
|-------|--------|--------|
| `pr_number` | PR number (required) | Which PR to review |
| `model` | `sonnet` (default) / `opus` / `haiku` | Choose the model |
| `scope` | `code errors` (default) / `full` | `full` runs all review agents; default is code quality + error handling |
| `force` | `false` (default) / `true` | Run despite the per-PR review cap (3) |
````

- [ ] **Step 3: Update the `full` explanation**

In the "What `full` changes" section, update the mechanism description: `full` is now the `scope` **input** choice, which the caller maps to `scope = "all"` (default maps to `"code errors"`) before passing it to the review command. Remove references to it being a comment keyword. Keep the "some repo-specific review commands ignore scope" note.

- [ ] **Step 4: Update the "Adopt in a repo" section and add migration note**

In `## Adopt in a repo`, update the pin guidance to `@v2.0.0`. Add a migration subsection:

```markdown
### Migrating from v1 (comment trigger) to v2 (workflow_dispatch)

v2 is a breaking change: the `@claude` comment trigger is replaced by
`workflow_dispatch`. To migrate a repo:

1. Replace its `.github/workflows/claude-review.yml` with the current
   `caller-stub` template (pinned to `@v2.0.0`).
2. Trigger reviews via the Actions tab or `gh workflow run` instead of commenting
   `@claude`.

Repos on `@v1.1.0` keep working unchanged until migrated.
```

- [ ] **Step 5: Validate README has no stale `@claude`-comment usage**

Run:

```bash
grep -nE '@claude|issue_comment|comment on a PR' README.md
```

Expected: matches only inside the migration note (explaining what v1 did) — verify each remaining hit is historical/migration context, not current instructions. Fix any that describe current usage.

- [ ] **Step 6: Commit**

```bash
git add README.md
git commit -m "docs: document workflow_dispatch usage and v1->v2 migration

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Release v2.0.0

**Files:**
- No file changes — tagging only.

**Interfaces:**
- Consumes: the merged Tasks 1-3 on the default branch.
- Produces: immutable tag `v2.0.0` that caller stubs pin to.

> This task runs only after Tasks 1-3 are merged to the default branch via the repo's normal PR process. Follow the README's "Releasing a new version" process for exact steps; the commands below are the standard immutable-tag release.

- [ ] **Step 1: Confirm the README release process**

Read the "Releasing a new version" section of `README.md` and follow it if it differs from the standard steps below.

- [ ] **Step 2: Tag and push (standard immutable-tag release)**

From the up-to-date default branch:

```bash
git tag -a v2.0.0 -m "v2.0.0: workflow_dispatch trigger (breaking); replaces @claude comment trigger"
git push origin v2.0.0
```

- [ ] **Step 3: Verify the tag resolves**

```bash
git ls-remote --tags origin v2.0.0
```

Expected: one line showing the `v2.0.0` ref.

---

## Notes for the implementer

- There are no unit tests in this repo; verification is YAML validity + targeted greps as shown. A real end-to-end test requires a consuming repo on `@v2.0.0` and is part of the (out-of-scope) migration.
- Keep the shell-injection-safe `env:`-passing pattern for every `gh` call. `github.actor` and `inputs.*` are trusted-enough to interpolate for logging, but prefer `env:` for anything written back.
- The `gh pr comment` URL→id extraction (`${URL##*-}`) depends on `gh` printing the comment URL to stdout; this is its documented behaviour. If a future `gh` version changes it, capture the id via `gh api` instead.
