# Provable-Games/.github

Org defaults and **shared CI workflows** for Provable-Games repositories.

> **Reusable workflows can't self-trigger.** Each caller repo keeps a thin
> stub with the real `on:` trigger + a `uses:` line. To run a shared workflow
> across all/new repos with no per-repo file, enforce it via an org
> **Repository Ruleset** ("require workflows to pass before merging") instead.

## Reusable npm publish

[`.github/workflows/publish-npm.yml`](.github/workflows/publish-npm.yml) builds
and publishes an npm package (bun install + typecheck + build + `npm publish`).

Caller `.github/workflows/publish.yml`:

```yaml
name: Publish to npm
on:
  release:
    types: [published]
  workflow_dispatch:
jobs:
  publish:
    uses: Provable-Games/.github/.github/workflows/publish-npm.yml@v1
    secrets: inherit
```

Requires the `NPM_TOKEN` secret. Inputs (all optional): `bun_version`,
`node_version`, `registry_url`, `access` (default `public`), `run_typecheck`.
Only needs the read-only default token, so no caller `permissions` block.

## Reusable code review

[`.github/workflows/code-review.yml`](.github/workflows/code-review.yml) runs
matrixed Claude + Codex reviews on pull requests. All generic logic lives here
(matrix build, prompt assembly, comment publishing, claude transcript
extraction). Each repo supplies only its own
review config and prompts.

### Adopt it in a repo

1. Add a thin caller workflow `.github/workflows/pr-review.yml`:

   ```yaml
   name: PR Review
   on:
     pull_request:
       # edited: re-review a corrected title, body or base.
       types: [opened, synchronize, reopened, ready_for_review, edited]
   jobs:
     review:
       uses: Provable-Games/.github/.github/workflows/code-review.yml@v1
       # A called workflow's job permissions can't exceed the caller's.
       permissions:
         contents: read
         issues: write
         pull-requests: write
         id-token: write
       secrets:
         CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
         CODEX_AUTH_DOT_JSON: ${{ secrets.CODEX_AUTH_DOT_JSON }}
   ```

   `id-token: write` lets the workflow find its own revision, so it fetches the
   review instructions from that same commit.

   Optional inputs: `config_path` (default `.github/review-agents.json`),
   `enable_claude` / `enable_codex` (default `true`),
   `block_on_severity` (default `false`), and `setup_script` (default empty):
   a script in your repo, run after checkout and before each review, that
   installs tools reviewers may need for targeted checks, such as Scarb or
   pnpm. It runs in its own step with a 2-minute limit, so add any install
   directory to the path for the review step with
   `echo "$dir" >> "$GITHUB_PATH"`.

   **`block_on_severity`** turns review into a merge gate: the review job
   fails when a reviewer reports a `[CRITICAL]` or `[HIGH]` finding. Off by
   default — reviews are advisory unless a repo opts in:

   ```yaml
   jobs:
     review:
       uses: Provable-Games/.github/.github/workflows/code-review.yml@v1
       with:
         block_on_severity: true
       secrets: inherit
   ```

   The review comment is posted either way, so a blocking finding is always
   visible before the job goes red. A reply of `Review incomplete: <reason>`
   also fails the job. A review that errored or produced no output does not
   block on its own.

2. Add `.github/review-agents.json` describing what to review:

   ```json
   {
     "agents": [
       {
         "agent_id": "sdk",
         "agent_name": "SDK Reviewer",
         "prompt_file": ".github/prompts/sdk-review.md",
         "diff_paths": ["src/", "tests/", "package.json"]
       }
     ]
   }
   ```

   An agent runs only when the PR touches one of its `diff_paths`. Add more
   agents for different areas (each gets its own scoped prompt + PR comment).
   Every completed reviewer run posts a new comment, titled
   `{model}-{effort} Code Review: [<short SHA>](<reviewed commit URL>)`.

   `diff_paths` and `exclude_paths` are matched as **literal path prefixes,
   not globs**: `"packages/"` matches `packages/foo/bar.cairo`, and
   `"package.json"` matches only that file. Do not write `"src/**"` — `**` is
   not special here, so the entry would match nothing. Use `"."` (or `""`) for
   a repo-root catch-all that matches every changed file.

   Optional **`exclude_paths`** narrows an agent to "matched by `diff_paths`
   but not under these paths" — useful for a catch-all `general` agent that
   reviews everything *except* the app dirs:

   ```json
   {
     "agent_id": "general",
     "agent_name": "General Reviewer",
     "prompt_file": ".github/prompts/general.md",
     "diff_paths": ["."],
     "exclude_paths": ["contracts/", "client/", "indexer/", "api/"]
   }
   ```

3. Add the prompt file(s) referenced above (`.github/prompts/sdk-review.md`),
   containing the review criteria for that scope.

### Review output format

Every review uses one org-wide format, which this workflow appends after the
repository prompt, so caller prompts hold review criteria only and define no
output format of their own. A clean review is exactly `lgtm`. Otherwise each
finding is:

```
### [SEVERITY] path/to/file.ext:LINE — what is wrong
**Likelihood:** Likely|Possible|Rare · **Effort:** Small|Medium|Large · **Do:** This PR|Follow-up · **Origin:** Introduced|Exposed|Pre-existing · **Evidence:** Verified|Inferred

- **Trigger:** what causes it.
- **Impact:** what breaks, for whom.
- **Fix:** the concrete change.
```

A review with findings ends with `**Complexity:** Low|Medium|High — why`.
`Do` says whether to fix it in this PR or in a follow-up issue or PR. Beyond
the repository's own criteria, every review checks that tests cover edge and
extreme cases, reports only significant simplifications, flags a PR
description that doesn't match the diff, and for smart contracts looks for
significant gas and CASM-size savings. A review
that cannot run because review material is missing replies
`Review incomplete: <reason>`. The full contract, with the severity,
likelihood, effort and `Do` definitions, is
[`.github/prompts/review-instructions.md`](.github/prompts/review-instructions.md),
which both reviewers get after the repository prompt.

### Secrets (org-level, inherited)

- `CLAUDE_CODE_OAUTH_TOKEN` — for the Claude reviewer.
- `CODEX_AUTH_DOT_JSON` — `~/.codex/auth.json` contents for the Codex reviewer.

Reviews are **skipped on fork PRs** (secrets aren't available to forks, so a
review would just fail/post empty). Same-repo and same-org branch PRs run
normally.

### Reviewer access and runners

Reviewers have full tool access so they can investigate: Claude Code runs with
`Bash,Read,Edit,Write,Glob,Grep,Agent` under `--permission-mode dontAsk`, and
Codex runs without its sandbox. They read any file for context, inspect history
and run targeted commands or single tests; CI runs the full checks, so they
don't rerun them. The caller repo's `CLAUDE.md`, settings, hooks and MCP servers
are not loaded. The accepted risk is that a write-access author can reach a
review credential; the response is to revoke and rotate it.

Both reviewers run on `ubuntu-slim` with a 15-minute job limit: reviews mostly
wait on the model API. Slim's container cannot run Codex's sandbox, which is why
Codex runs without it.

### Reviewer models and effort

Model and effort come from org Actions variables, and both CLIs install at
their latest release, so upgrading every caller's reviewers is one variable
change. The Claude Code Action's moving `v1` tag installs its own current
Claude Code.

| Variable | Required | Purpose |
|---|---|---|
| `CLAUDE_REVIEW_MODEL`, `CLAUDE_REVIEW_EFFORT` | for Claude | `--model` and `--effort` |
| `CODEX_REVIEW_MODEL`, `CODEX_REVIEW_EFFORT` | for Codex | `model` and `model_reasoning_effort` |
| `CODEX_CLI_VERSION` | no | `@openai/codex` version, default `latest` |
| `CODEX_REVIEW_TIMEOUT_SECONDS` | no | Timeout for each Codex review, capped by the 12-minute review step |

A reviewer whose model or effort variable is empty fails with an error rather
than falling back to a stale default. If that happens, give the repository
access to the org variables.

### Versioning

Callers pin to a tag (`@v1`). Cut a new tag when the workflow changes; move
`v1` forward for backward-compatible updates.
