# Provable-Games/.github

Org defaults and **shared CI workflows** for Provable-Games repositories.

## Reusable code review

[`.github/workflows/code-review.yml`](.github/workflows/code-review.yml) runs
matrixed Claude + Codex reviews on pull requests. All generic logic lives here
(matrix build, prompt assembly, the codex sandbox workaround, comment
upserting, claude transcript extraction). Each repo supplies only its own
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
   `enable_claude` / `enable_codex` (default `true`).

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

3. Add the prompt file(s) referenced above (`.github/prompts/sdk-review.md`),
   containing the review criteria for that scope.

### Secrets (org-level, inherited)

- `CLAUDE_CODE_OAUTH_TOKEN` — for the Claude reviewer.
- `CODEX_AUTH_DOT_JSON` — `~/.codex/auth.json` contents for the Codex reviewer.

### Variables (org-level, optional — sensible defaults baked in)

| Variable | Default | Purpose |
|---|---|---|
| `CLAUDE_REVIEW_MODEL` | `claude-opus-4-8` | Claude review model |
| `CODEX_REVIEW_MODEL` | `gpt-5.5` | Codex review model |
| `CODEX_REVIEW_EFFORT` | `xhigh` | Codex reasoning effort |
| `CODEX_CLI_VERSION` | `latest` | `@openai/codex` npm spec |

A model deprecation is a one-place change here (org variable), not an edit in
every repo.

### Versioning

Callers pin to a tag (`@v1`). Cut a new tag when the workflow changes; move
`v1` forward for backward-compatible updates.
