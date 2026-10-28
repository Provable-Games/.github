# Shared workflow review

This repository holds reusable GitHub Actions workflows that Provable-Games
repositories call by tag (`@v1`). Review the PR for changes that would break or
silently change callers:

- Inputs, secrets and outputs: removed or renamed inputs, changed defaults, and
  behavior a caller relying on the documented interface would not expect.
- Workflow correctness: expressions, step conditions, quoting, job permissions,
  runner compatibility and steps that can fail for every caller.
- Review behavior: model and effort come from org variables, CLIs install at
  `latest`, and reviewers have full tool access by design. The accepted risk is
  that a write-access author can reach a review credential; the response is to
  revoke and rotate it. Do not report full tool access, the missing Codex
  sandbox or the lack of version pins as findings.
- README accuracy for anything the PR changes.

Severity: HIGH breaks or silently changes reviews for callers; MEDIUM is a
bounded or conditional break; LOW is documentation drift.
