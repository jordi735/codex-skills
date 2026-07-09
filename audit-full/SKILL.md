---
name: audit-full
description: "Thorough nested-agent audit of the current project for dead code, YAGNI, DRY, KISS violations, and bugs. A root orchestrator discovers domains, domain coordinators fan out to specialist child agents, verifiers confirm findings against source, and the root writes a ranked /tmp/{project}-audit-report.md with file:line references, suggested fixes, reasons, and 1-10 confidence scores. Use when the user asks for a deep code audit, full project review, or runs /audit."
---

# Audit

Launch a thorough, multi-stage project audit via nested Codex gpt-5.6-sol ultra subagents. Produces a single ranked fix sheet at `/tmp/<project>-audit-report.md`. Does NOT modify any source code — the deliverable is a report the user applies manually.

---

## Invocation

- `/audit` → infer `PROJECT_NAME` from `basename "$(pwd)"`
- `/audit <name>` → use explicit name

## Setup

1. Determine `PROJECT_NAME` per invocation rules above.
2. Set `REPORT_PATH=/tmp/${PROJECT_NAME}-audit-report.md`.
3. If `REPORT_PATH` already exists, overwrite it during final synthesis.
4. Track progress with one task per pipeline step: Discovery, Domain Audit, Synthesis.

---

## Reality Filter

This audit targets CRITICAL findings — bugs and quality issues with realistic, user-visible harm. It explicitly rejects "niceness" findings (input-validation polish, error-message formatting, defensive coding for unreachable states, generic style improvements).

A finding is valid only if ALL of these are true:

1. **Realistic path** — manifests in normal usage, not adversarial input or theoretical edge cases.
2. **User-visible harm** — data loss, wrong output, crash, security breach, broken feature, or a quality issue that has already produced a concrete bug.
3. **Actually wrong** — not "could be cleaner / safer / more defensive".

Reject explicitly:
- Missing input validation when the system handles bad input safely (any error response counts — formatting and status codes are NOT bugs).
- Error-message wording, formatting, or HTTP status code polish.
- Defensive coding for states that cannot occur given the call sites.
- Hardening, robustness, or future-proofing suggestions.
- Style, naming, readability, comment quality, "could be more idiomatic".
- Interface members, exported library API, and platform-required hooks (not "dead code").
- Coincidental duplication where two functions look alike but mean different things.

Tell: if a finding's `Why` field would naturally contain "could", "might", "should be more", or "would be safer" — drop it.

The filter is inlined into every subagent prompt; the verifier re-applies it to drop niceness findings that slipped through; final synthesis drops the rest by confidence and severity thresholds.

---

## Pipeline

### 1. Domain Discovery

The root agent maps the project into logical domains by inspecting the repo.
Do not write `domains.json` by default.

Explore the codebase and identify logical domains — bounded contexts,
feature areas, or coherent top-level modules. A domain is a cohesive
chunk of code addressing one concern (e.g. "auth", "billing",
"notifications", "search-indexer"). Group related folders where it
makes sense; not every folder is a domain.

For each domain, produce in memory:
- `name`: kebab-case identifier
- `paths`: list of relevant source paths or glob patterns
- `description`: one-sentence purpose
- `approx_loc`: approximate lines of production code

Ignore vendored dependencies, build output, lockfiles, generated code,
node_modules, .git, .next, dist, build, coverage, and test files
(unit / integration / e2e tests, fixtures, mocks). Files whose names
merely contain "test" as a feature term — like an A/B-testing module —
are production code and must be included.

### 2. Per-Domain Coordination

Launch one domain coordinator gpt-5.6-sol ultra subagent per domain, in parallel. Each
coordinator owns exactly one domain.

Each domain coordinator should:
- inspect only the assigned domain paths, plus cross-references needed to verify usage
- launch specialist child gpt-5.6-sol ultra agents in parallel
- wait for specialist results
- launch one verifier/merger child for the entire domain
- return only the verified domain findings packet to the root agent

Domain coordinators must not modify source code. Intermediate coordination
should happen through subagent final responses. Write intermediate files
only when a payload is too large to return cleanly, then include the path
in the parent response.

### 3. Specialist Auditors

Within each domain coordinator, launch specialist child gpt-5.6-sol ultra agents for:
- correctness bugs
- code quality findings covering DEAD, YAGNI, DRY, and KISS

For very large domains, the coordinator may split quality into separate
category agents, but the default is one bug hunter and one quality hunter.

Specialists must skip test files (unit / integration / e2e tests, fixtures,
mocks). Files whose names merely contain "test" as a feature term — like
an A/B-testing module — are production code and MUST be audited.

**Code Quality scope:**

Look for these four categories ONLY, AND ONLY when they cause concrete harm:
- DEAD   : functions, variables, imports, exports with NO consumers anywhere.
           Must be safe to delete. Do NOT flag interface members, exported
           library API, or platform-required hooks.
- YAGNI  : speculative code that is actively in the way — confuses readers,
           blocks changes, or has decayed into inconsistent state. Pure
           unused-yet-harmless abstractions are NOT findings.
- DRY    : duplicated logic that HAS caused or IS causing divergent behavior
           (one copy was fixed and the other was not; copies disagree).
           "Could be extracted" without concrete harm is NOT a finding.
- KISS   : over-engineering that is actively producing bugs or blocking
           changes. "Could be simpler" without concrete harm is NOT a finding.

Do NOT flag bugs, correctness issues, or missing error handling in the
quality audit.

**Bug Hunt scope:**

Look for bugs with concrete failure modes:
- Null / undefined dereferences where the value IS reachable as null
  (not "could theoretically be null").
- Off-by-one errors with a concrete failing input.
- Race conditions with a concrete interleaving that produces wrong state.
- Inverted conditions, wrong operators, bad boundary checks.
- Resource leaks on hot paths (not "this rarely-called function might leak").
- Wrong logic that produces incorrect output for normal inputs.
- Swallowed errors that HIDE a real failure mode users will experience
  (not "we should log this").

Do NOT flag style, quality, or design issues in the bug hunt.

Each specialist applies the Reality Filter before returning findings.
If a finding's `Why` uses "could", "might", "should be more", or "would
be safer", drop it.

For each finding, return this exact markdown block:

### [CATEGORY] short title
- **File**: `path/to/file.ext:line` or `path/to/file.ext:start-end`
- **Severity**: HIGH | MED
- **Status**: unverified
- **What**: one-line description of the issue
- **Why**: concrete user-visible harm or concrete divergent-bug history
- **Fix**: concrete suggested change

Severity guide (HIGH and MED only — no LOW tier):
- HIGH: actively causing a bug, divergence, crash, security issue, data loss,
        wrong output, broken feature, or blocking concrete work right now.
- MED:  realistic problem under specific conditions, lower blast radius.

Specialists return findings directly to the domain coordinator. They should
not write files unless the response would be too large.

### 4. Domain Verify + Merge

Each domain coordinator launches one verifier/merger child after all
specialists finish.

For EACH finding, the verifier must:
1. Open the referenced source file(s) and read the cited lines plus
   surrounding context.
2. Verify the CLAIM: does the issue exist exactly as described?
3. Apply the REALITY FILTER:
   - Realistic execution path? (not adversarial / impossible / theoretical)
   - Concrete user-visible harm? (data loss, wrong output, crash, security,
     broken feature, OR a quality issue that has produced a divergent bug)
   - Actually wrong, not "could be cleaner / safer / more defensive"?
   Mark as `false-positive` (with a one-line note in `Why`) when:
   - The finding's `File` is a test file (unit / integration / e2e test,
     fixture, or mock) — not a production file that merely has "test" in
     its name as a feature term.
   - Missing input validation but the system handles bad input safely.
   - Error-message formatting, wording, or HTTP status code polish.
   - Defensive coding for unreachable states; hardening; future-proofing.
   - Style, naming, readability, "could be more idiomatic".
   - Interface contract members flagged as DEAD.
   - Coincidental duplication where the two functions mean different things.
4. Update `Status`:
   - `confirmed`      — claim accurate AND passes Reality Filter.
   - `false-positive` — claim wrong OR fails Reality Filter.
   - `uncertain`      — needs human judgment about realistic harm.
5. Assign confidence on a 1-10 scale:
   - 9-10: verified from source AND clearly causes the harm described
   - 7-8 : verified, harm is plausible
   - 4-6 : claim partially true OR harm uncertain
   - 1-3 : false-positive (claim wrong) or no realistic harm
6. Adjust `Severity` if source reveals different impact (HIGH or MED only —
   if the only honest severity is "minor", it is a `false-positive`).
7. Refine `Fix` to a precise concrete change.

Then DEDUPE across specialist results: if the same underlying issue at the
same file:line appears more than once, merge into a single entry. Keep the
more specific category; if genuinely multi-faceted, list both categories
separated by `+` (e.g. `[YAGNI+DEAD]`).

The verifier returns this domain packet:

```markdown
# Verified Findings: <domain.name>
Confirmed: X | Uncertain: Y | False-positive: Z

### [CATEGORY] short title
- **File**: `path/to/file.ext:line`
- **Severity**: HIGH | MED
- **Confidence**: N/10
- **Status**: confirmed | false-positive | uncertain
- **What**: ...
- **Why**: ...
- **Fix**: ...
```

Keep ALL findings in the domain packet, including false positives. The root
agent drops them during final synthesis.

### 5. Final Synthesis

The root agent merges all verified domain packets and writes:

`/tmp/<PROJECT_NAME>-audit-report.md`

Use this structure:

```markdown
# Audit Report: <PROJECT_NAME>

Generated: <ISO-8601 date>
Domains audited: N
Findings: X confirmed | Y uncertain | Z false-positive (dropped)

## Summary by category

| Category | HIGH | MED | Total |
|----------|------|-----|-------|
| BUG      | ...  | ... | ...   |
| DEAD     | ...  | ... | ...   |
| DRY      | ...  | ... | ...   |
| KISS     | ...  | ... | ...   |
| YAGNI    | ...  | ... | ...   |

## Findings

Sorted by priority = severity_weight * confidence, descending.
severity_weight: HIGH=3, MED=2.

DROP rules:
- DROP every `false-positive` entry.
- DROP every entry with Confidence < 6.
- DROP every `uncertain` entry whose Severity is not HIGH.
- DROP any LOW-severity entry that slipped through.

For each remaining finding:

### [CATEGORY] <domain.name> — short title
- **File**: `path/to/file.ext:line`
- **Severity**: HIGH | MED
- **Confidence**: N/10
- **Status**: confirmed | uncertain
- **What**: ...
- **Why**: ...
- **Fix**: ...
```

The root agent returns the absolute path to `REPORT_PATH` plus a 3-line
summary:
- total confirmed findings
- HIGH-severity count
- top 3 categories by total

---

## Final Output

After final synthesis completes:

1. Print the absolute path of `REPORT_PATH`.
2. Print the 3-line summary.
3. Suggest: "Open the report and apply fixes manually. Re-run `/audit` any time to regenerate."

## Notes

- **No source files are modified** at any step.
- The only required file output is `/tmp/<PROJECT_NAME>-audit-report.md`.
- Intermediate files are optional overflow artifacts only.
- Prefer nested delegation over flat orchestration: root owns project synthesis, domain coordinators own domain-level fan-out, verifiers own confirmation and dedupe.
- Launch sibling agents in parallel whenever their work is independent.
- If a subagent fails, its parent records which role failed and continues with available findings. The final report must note partial audit coverage.
- Each subagent prompt must be self-contained and include the Reality Filter, plus concrete values for `<cwd>`, `<PROJECT_NAME>`, and any `<domain.*>` placeholders.
- All agents must be gpt-5.6-sol ultra
