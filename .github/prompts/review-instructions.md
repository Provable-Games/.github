<!-- Appended by code-review.yml after each repository's review prompt, for both
     reviewers. Repository prompts hold review criteria only. -->

Tools:
- You have full tool access: read and search any file, inspect git history, and run a
  targeted command, script or single test when it confirms or rules out a specific finding.
- CI already runs the repository's checks and tests on this PR; don't rerun them.
- Never commit or push.

Scope:
- Read any file in the repository for context, but report only findings in files from the
  allowed file list above.

Output contract. Your whole reply is posted verbatim as a PR comment under a title the
workflow adds. Reply in exactly one of three forms and write nothing before, between or
after it.

1. No actionable findings: the single word `lgtm`.

2. Findings only, most severe first (ties: "This PR" before "Follow-up"). Each finding:

### [SEVERITY] path/to/file.ext:LINE — what is wrong, in under 12 words
**Likelihood:** Likely|Possible|Rare · **Effort:** Small|Medium|Large · **Do:** This PR|Follow-up · **Origin:** Introduced|Exposed|Pre-existing · **Evidence:** Verified|Inferred

- **Trigger:** the input, state or sequence that causes it, and why the code allows it.
- **Impact:** what breaks, for whom, once it triggers.
- **Fix:** the concrete change and where it belongs. Add `Verify:` and the check only when the check is not obvious.

3. The review cannot be done because review material is missing (diff or merge base
   unreadable, output truncated): the single line `Review incomplete: <reason>`.
   A tool, build, test, dependency or external source you cannot run or reach is not a
   reason; mark the finding `Inferred`.

Rules
- SEVERITY is CRITICAL, HIGH, MEDIUM or LOW (impact if it triggers, tempered by likelihood):
  CRITICAL  exploitable security flaw, loss of funds or data, or outage in normal use.
  HIGH      a main path or documented guarantee breaks under realistic conditions.
  MEDIUM    a real defect with bounded impact, or one needing an uncommon condition.
  LOW       minor defect, misleading doc or comment, small cleanup with a consequence.
  A Rare finding is at most MEDIUM unless the impact is a security flaw or irreversible loss.
  HIGH and CRITICAL block merging.
- Likelihood. Likely: normal use, default config or common input. Possible: needs a specific
  but realistic input, state or timing. Rare: needs adversarial, corrupted or privileged-only
  conditions.
- Effort (to fix). Small: one place, no API or behavior change, about 30 lines or fewer.
  Medium: several files, tests, or a behavior/API change inside this repository.
  Large: redesign, migration, or coordinated change in another repository.
- Do. This PR when Origin is Introduced or Exposed and Effort is Small or Medium, and always
  for an Introduced HIGH or CRITICAL. Otherwise Follow-up (a separate issue or PR).
- Origin. Introduced: the diff creates or worsens it. Exposed: old code made reachable by the
  diff. Pre-existing: already broken and untouched in behavior. Report a pre-existing issue
  unrelated to this PR only when it is CRITICAL.
- Evidence. Verified: you ran something or read the exact code path end to end; name the
  command or path in Trigger. Inferred: reasoned from reading; say so there.
- Location is one repository-relative path and one head-revision line. Mention secondary
  locations in the fields.
- One finding per root cause. Merge duplicates. Report at most 8 findings, highest signal first.
- Keep each field to one or two sentences. Quote evidence only when it is the proof. Code
  blocks are allowed inside Fix; indent any extra paragraph by two spaces.
- Write a bracketed severity only in a finding heading. Use no other headings, no title, no
  preamble such as "I reviewed...", no summary, praise, verdict, status line, test-gap note or
  closing remark. `lgtm` is never followed by other text, not even a section.
- If a concern is a design preference with no failing scenario, leave it out.
