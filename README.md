# magiqsoftware/claude-review

Org-wide **reusable** GitHub Actions workflow for on-demand Claude PR reviews.
Comment `@claude` on a pull request and a multi-agent code review is posted back
as a PR comment. The logic lives here once; consuming repos add a ~12-line stub.

## One-time org setup

1. **Organisation secret** `ANTHROPIC_API_KEY` (Anthropic API key, pay-per-token).
   Settings → Secrets and variables → Actions → *Organization secrets*. Scope its
   visibility to the repos that should have Claude review. One key = one shared
   Anthropic spend cap; use separate keys per team if you want separate budgets.
2. **Organisation variable** `CLAUDE_REVIEW_USERS` = JSON array of allowlisted
   GitHub logins, e.g. `["plimmerd","alice"]`. A repo may override with a repo
   variable of the same name.
3. **Reusability access** — this repo is private, so enable cross-repo calls once:
   Settings → Actions → General → *Access* → "Accessible from repositories owned by
   the organization". (No secrets live in this workflow, so private vs public has no
   bearing on key safety — the API key is injected at runtime from the org secret.)
4. **Releases are immutable tags** — each release is a fresh `vMAJOR.MINOR.PATCH`
   tag (see *Releasing a new version* below). Consumers pin to an exact tag, never a
   moving alias, so an upstream change only reaches a repo via a reviewed bump there.

## Adopt in a repo

Copy `caller-stub/.github/workflows/claude-review.yml` into the target repo's
`.github/workflows/`. Pin the `uses:` line to a specific release tag — e.g.
`@v1.0.0` (immutable), **not** `@v1` (moving). Nothing else — credentials and
allowlist inherit from the org.

## Usage (comment on a PR)

| Comment | Effect |
|---------|--------|
| `@claude` | Default review — Sonnet, scoped to code quality + error handling |
| `@claude full` | All review agents (comments, tests, errors, types, code, simplify) |
| `@claude opus` / `sonnet` / `haiku` | Choose the model |
| `@claude opus full` | Combine (keywords are order-independent) |
| `@claude force` | Run despite the per-PR review cap |

## Limits (and why)

- **3 reviews per PR** — reviews cost API tokens; the cap stops runaway spend.
  Override with `force`.
- **Default Sonnet / code+errors** — cheapest useful default; `opus` and `full`
  are opt-in.
- **`@claude` only (not `/review`)** — `/review` collides with the Qodo bot.
- **Allowlist + write access** — only trusted users can spend tokens / use secrets.
- **Fork PRs refused** — the workflow runs with secrets, so it won't run untrusted
  fork code.

## Releasing a new version

A change to the workflow ships as a **new immutable tag**; each consuming repo then
bumps to it in a reviewed PR. That bump is the review checkpoint — because the
caller job runs with the consuming repo's secrets, upstream changes must not take
effect silently.

1. Merge the change to `main` (PR + review; `main` is branch-protected).
2. Cut the next semver tag — patch for fixes, minor for new options, major for
   breaking changes:
   ```sh
   git tag v1.0.1            # tags current main HEAD; or: git tag v1.0.1 <commit>
   git push origin v1.0.1
   ```
   Release tags are immutable — the `v*` ruleset blocks moving or deleting them.
   **Never** `git tag -f` an existing tag; always cut a new one.
3. In each consuming repo, bump the `uses: …@vX.Y.Z` pin to the new tag in a
   reviewed PR.

## Versioning notes

- Consumers **pin to an exact `@vX.Y.Z`**, never a moving alias like `@v1` — a
  moving ref would let upstream changes execute with the consumer's secrets without
  review in that repo.
- The Claude Code CLI is pinned inside the workflow (`CLAUDE_CODE_VERSION`); bump it
  deliberately as part of a release.
- The `pr-review-toolkit` marketplace plugin tracks latest (no version pin available).
