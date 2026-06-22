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
4. **Tagging** — keep a moving `v1` tag on the latest compatible commit so callers
   pinning `@v1` get fixes automatically. Re-point it after each release:
   `git tag -f v1 && git push -f origin v1`.

## Adopt in a repo

Copy `caller-stub/.github/workflows/claude-review.yml` into the target repo's
`.github/workflows/`. Nothing else — credentials and allowlist inherit from the org.

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

## Versioning

Callers pin `@v1` (moving major) or a commit SHA (frozen). The Claude Code CLI is
pinned inside the workflow (`CLAUDE_CODE_VERSION`) for reproducibility; bump it
deliberately. The `pr-review-toolkit` marketplace plugin tracks latest.
