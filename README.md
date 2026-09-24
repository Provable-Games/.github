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
(matrix build, prompt assembly, the codex sandbox workaround, comment
publishing, claude transcript extraction). Each repo supplies only its own
review config and prompts.

### Adopt it in a repo

1. Add a thin caller workflow `.github/workflows/pr-review.yml`:

   ```yaml
   name: PR Review
   on:
     pull_request:
       types: [opened, synchronize, reopened, ready_for_review]
   jobs:
     review:
       uses: Provable-Games/.github/.github/workflows/code-review.yml@v1
       secrets: inherit
   ```

   Optional inputs: `config_path` (default `.github/review-agents.json`),
   `enable_claude` / `enable_codex` (default `true`), and
   `block_on_severity` (default `false`).

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
   visible before the job goes red. A review that errored or produced no
   output cannot match, so it never blocks on its own.

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

### Secrets (org-level, inherited)

- `CLAUDE_CODE_OAUTH_TOKEN` — for the Claude reviewer.
- `CODEX_AUTH_DOT_JSON` — `~/.codex/auth.json` contents for the Codex reviewer.

Reviews are **skipped on fork PRs** (secrets aren't available to forks, so a
review would just fail/post empty). Same-repo and same-org branch PRs run
normally.

### Reviewer models and effort

The shared workflow pins these settings so organization and repository Actions
variables cannot change the requested reviewer model or effort:

| Reviewer | Model | Effort | CLI |
|---|---|---|---|
| Claude | `claude-opus-5-5` | `medium` | `@anthropic-ai/claude-code@2.1.281` |
| Codex | `gpt-6-luna` | `max` | `@openai/codex@0.155.0` |

Claude CLI 2.1.281 supports Opus 5.5; the reusable workflow passes this pinned
binary to the Claude Code Action.

### Optional Actions variables

| Variable | Default | Purpose |
|---|---|---|
| `CODEX_REVIEW_TIMEOUT_SECONDS` | unset | Optional timeout for each Codex review |

Model and effort changes are made in this shared workflow and its documentation.

### Versioning

Callers pin to a tag (`@v1`). Cut a new tag when the workflow changes; move
`v1` forward for backward-compatible updates.
