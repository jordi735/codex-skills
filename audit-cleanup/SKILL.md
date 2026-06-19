---
name: audit-cleanup
description: "Behavior-preserving project audit for product conventions, cleanup, clean code standards, and module-split candidates. Use when the user asks for a product-convention review, cleanup audit, clean-code standards review, module extraction review, refactor candidate list, consistency pass, non-functional improvement report, or any codebase cleanup that must not change existing behavior."
---

# Product Cleanup Audit

Launch a thorough, multi-stage project audit focused on product convention,
cleanup, clean code standards, and behavior-preserving module splits. Produces
a single ranked report at
`/tmp/<project>-product-cleanup-audit.md`.

This skill does not modify source code. The deliverable is an evidence-backed
report of behavior-preserving improvements.

---

## Invocation

- `$product-cleanup-audit` or "product cleanup audit" -> infer
  `PROJECT_NAME` from `basename "$(pwd)"`
- `/cleanup-audit` -> infer `PROJECT_NAME` from `basename "$(pwd)"`
- `/cleanup-audit <name>` -> use explicit name

## Setup

1. Determine `PROJECT_NAME` per invocation rules above.
2. Set `REPORT_PATH=/tmp/${PROJECT_NAME}-product-cleanup-audit.md`.
3. If `REPORT_PATH` already exists, overwrite it during final synthesis.
4. Track progress with one task per pipeline step:
   Discovery, Domain Audit, Verification, Synthesis.

---

## Non-Functional Contract

No source files are modified at any step.

Every actionable recommendation must preserve existing functionality. A fix is
behavior-preserving only when it leaves these unchanged:

- public APIs, routes, schemas, persistence formats, auth/permission rules
- business rules, validation semantics, default values, feature flags
- state transitions, ordering, timing assumptions, side effects
- telemetry meaning and analytics event semantics
- user-visible product promises

User-visible copy, layout, and interaction findings are allowed only when the
change aligns with a clearly established local product convention and does not
change meaning or workflow. If a recommendation needs product/UX judgment, mark
it `decision-only` and keep it out of the actionable fix list.

---

## Evidence Filter

A finding is valid only if ALL of these are true:

1. **Convention evidence** - it cites an existing repo convention, product
   standard, design system pattern, documented rule, or at least two local
   examples that establish the pattern.
2. **Behavior safety** - the proposed fix is concrete and can be made without
   changing runtime behavior, contracts, or product meaning.
3. **Cleanup value** - the issue creates real maintenance cost, convention
   drift, reader confusion, duplicated effort, or product inconsistency.
4. **Verification path** - the report can name the tests, snapshots, type
   checks, screenshots, or search results that would validate preservation.

Reject explicitly:

- personal taste, generic style preferences, or "more idiomatic" rewrites
- feature ideas, UX redesigns, information architecture changes, new product
  behavior, or copy changes that alter meaning
- broad architecture rewrites, new frameworks, dependency swaps, or speculative
  abstractions
- splits that require public API changes, new framework boundaries, dependency
  cycles, or reordered side effects
- formatter-only issues covered by existing automated formatting
- defensive coding for unreachable states
- deletion of exported API, platform hooks, migration code, public config, or
  integration points without proof they are private and unused
- coincidental duplication where similar code represents different product
  concepts

Tell: if the `Why` field naturally says "would be nice", "could be cleaner",
"might help later", or "should be more modern", drop the finding.

---

## Categories

Use these categories only:

- **PRODUCT-CONVENTION**: product terminology, visible states, action labels,
  navigation patterns, UI component usage, analytics naming, permission/feature
  naming, or workflow conventions that differ from established local practice.
- **CODE-CONVENTION**: module layout, file naming, imports/exports, service
  boundaries, hook/component patterns, error-handling shape, or async/state
  patterns that differ from established local practice without being bugs.
- **CLEANUP**: stale comments, obsolete TODOs, redundant config, dead private
  code, unused private assets, abandoned flags, or generated/build artifacts
  that can be safely removed.
- **CLEAN-CODE**: behavior-preserving simplifications that reduce nesting,
  indirection, confusing names, unnecessary branching, or local complexity
  while keeping interfaces and outputs stable.
- **MODULE-SPLIT**: code that should be extracted into its own file or module
  because it is a coherent private concern, feature-local concern, or reusable
  concern that local conventions already keep separate.
- **DRY**: duplicated implementation whose semantics are demonstrably identical
  and can be consolidated without coupling distinct product concepts.

## Module Split Filter

A `MODULE-SPLIT` finding is valid only when the proposed extraction:

- names the exact destination module or file shape implied by repo conventions
- preserves public APIs, import paths exposed outside the domain, side effects,
  initialization order, runtime behavior, and product meaning
- keeps dependencies acyclic and does not couple distinct product concepts
- reduces concrete reader confusion, ownership ambiguity, or repeated editing
  friction that is visible in the current code

Reject module-split findings when they only cite file length, "cleaner" taste,
possible future reuse, or a desire to move miscellaneous code into generic
`utils`, `helpers`, or `common` modules.

Do not report correctness bugs in this skill. If a bug is found, mention in
partial coverage notes that a separate bug audit is appropriate.

---

## Pipeline

### 1. Discovery

The root agent maps the project into logical domains and builds a lightweight
convention map by inspecting the repo. Do not write discovery artifacts by
default.

Inspect:

- product docs, README files, design-system docs, Storybook/configuration,
  route maps, copy constants, navigation definitions, analytics definitions
- production source, shared UI/components, API/controller/service boundaries,
  package conventions, lint/type/test configuration
- tests only as evidence for expected behavior or established conventions,
  unless the user explicitly asks to audit tests too

Ignore vendored dependencies, lockfiles, build output, generated code,
`node_modules`, `.git`, `.next`, `dist`, `build`, and `coverage`.

For each domain, produce in memory:

- `name`: kebab-case identifier
- `paths`: relevant source paths or glob patterns
- `description`: one-sentence purpose
- `approx_loc`: approximate production lines of code
- `convention_evidence`: notable local conventions with file references

### 2. Per-Domain Coordination

Launch one domain coordinator subagent per domain, in parallel when possible.
Each coordinator owns exactly one domain.

Each domain coordinator should:

- inspect only the assigned paths plus cross-references needed to verify
  conventions and behavior safety
- launch specialist child agents in parallel
- wait for specialist results
- launch one verifier/merger child for the entire domain
- return only the verified domain findings packet to the root agent

Domain coordinators must not modify source code. Intermediate coordination
should happen through subagent final responses. Write intermediate files only
when a payload is too large to return cleanly, then include the path in the
parent response.

### 3. Specialist Auditors

Within each domain coordinator, launch specialist child agents for:

- product convention audit
- cleanup and dead-artifact audit
- clean code, DRY, and code convention audit
- module boundary and extraction audit

For very large domains, split these categories further by path or concern.

Each specialist must apply the Non-Functional Contract and Evidence Filter
before returning findings. The module boundary specialist must also apply the
Module Split Filter.

For each finding, return this exact markdown block:

```markdown
### [CATEGORY] short title
- **File**: `path/to/file.ext:line` or `path/to/file.ext:start-end`
- **Severity**: HIGH | MED | LOW
- **Status**: unverified
- **What**: one-line description of the issue
- **Evidence**: local convention evidence, with file references
- **Behavior safety**: why the proposed fix preserves functionality
- **Why**: concrete cleanup or convention value
- **Fix**: concrete suggested change
- **Validation**: tests/checks/searches/screenshots that would confirm behavior preservation
```

For `MODULE-SPLIT` findings, `Fix` must name the proposed target module or file,
the code to move, and the imports/exports or call sites that stay behaviorally
equivalent.

Severity guide:

- HIGH: widespread convention drift, unsafe-to-maintain cleanup debt, or
  confusing structure that repeatedly affects product consistency or developer
  changes; fix is still behavior-preserving.
- MED: localized but meaningful convention inconsistency, stale private code, or
  complexity that creates recurring maintenance cost.
- LOW: small, safe cleanup with strong evidence. Keep LOW findings sparse and
  avoid listing every minor preference.

### 4. Domain Verify + Merge

Each domain coordinator launches one verifier/merger child after all
specialists finish.

For EACH finding, the verifier must:

1. Open the referenced source file(s) and read cited lines plus surrounding
   context.
2. Open the cited convention evidence and verify it establishes the claimed
   pattern.
3. Search for consumers and side effects when a cleanup removes, moves, or
   consolidates code.
4. Apply the Non-Functional Contract:
   - Would the fix change runtime behavior, contracts, product meaning, data,
     routes, auth, validation, timing, or side effects?
   - If yes, mark `false-positive` or `decision-only`.
5. For `MODULE-SPLIT` findings, apply the Module Split Filter:
   - Does the target module shape match existing local conventions?
   - Is the extracted concern coherent and separable from surrounding code?
   - Do dependencies remain acyclic and ownership boundaries intact?
   - Do public import paths, side effects, and initialization order stay stable?
6. Apply the Evidence Filter:
   - Is this grounded in local convention evidence rather than taste?
   - Is the cleanup value concrete?
   - Is there a clear validation path?
7. Update `Status`:
   - `confirmed` - accurate, evidence-backed, and behavior-preserving
   - `decision-only` - plausible product convention issue, but needs human
     product/UX judgment before changing
   - `false-positive` - claim is wrong, preference-only, unsafe, or behavior
     changing
   - `uncertain` - source evidence is incomplete, but the issue may be real
8. Assign confidence on a 1-10 scale:
   - 9-10: verified from source, convention evidence is clear, fix is safely
     behavior-preserving
   - 7-8: verified, minor judgment remains
   - 4-6: claim partially true or behavior safety uncertain
   - 1-3: false-positive or no concrete convention/cleanup value
9. Adjust severity if source reveals different impact.
10. Refine `Fix` and `Validation` into precise concrete steps.

Then dedupe across specialist results. If the same issue appears more than
once, merge into a single entry. If genuinely multi-faceted, list categories
separated by `+`, such as `[PRODUCT-CONVENTION+CODE-CONVENTION]`.

The verifier returns this domain packet:

```markdown
# Verified Findings: <domain.name>
Confirmed: X | Decision-only: Y | Uncertain: Z | False-positive: W

### [CATEGORY] short title
- **File**: `path/to/file.ext:line`
- **Severity**: HIGH | MED | LOW
- **Confidence**: N/10
- **Status**: confirmed | decision-only | uncertain | false-positive
- **What**: ...
- **Evidence**: ...
- **Behavior safety**: ...
- **Why**: ...
- **Fix**: ...
- **Validation**: ...
```

Keep all findings in the domain packet, including false positives. The root
agent drops or separates them during final synthesis.

### 5. Final Synthesis

The root agent merges all verified domain packets and writes:

`/tmp/<PROJECT_NAME>-product-cleanup-audit.md`

Use this structure:

```markdown
# Product Cleanup Audit: <PROJECT_NAME>

Generated: <ISO-8601 date>
Domains audited: N
Findings: X confirmed | Y decision-only | Z uncertain | W false-positive (dropped)

## Convention Map

Brief bullets summarizing the strongest product/code conventions discovered,
with source references.

## Summary by category

| Category | HIGH | MED | LOW | Total |
|----------|------|-----|-----|-------|
| PRODUCT-CONVENTION | ... | ... | ... | ... |
| CODE-CONVENTION    | ... | ... | ... | ... |
| CLEANUP            | ... | ... | ... | ... |
| CLEAN-CODE         | ... | ... | ... | ... |
| MODULE-SPLIT       | ... | ... | ... | ... |
| DRY                | ... | ... | ... | ... |

## Actionable Findings

Sorted by priority = severity_weight * confidence, descending.
severity_weight: HIGH=3, MED=2, LOW=1.

DROP from this section:
- every `false-positive` entry
- every `decision-only` entry
- every entry with Confidence < 6
- every `uncertain` entry unless Severity is HIGH and Confidence >= 7
- every `MODULE-SPLIT` entry unless Status is `confirmed`, Confidence >= 6,
  and `Fix` names the target module plus stable imports/exports or call sites
- any entry whose fix is not clearly behavior-preserving

For each remaining finding:

### [CATEGORY] <domain.name> - short title
- **File**: `path/to/file.ext:line`
- **Severity**: HIGH | MED | LOW
- **Confidence**: N/10
- **Status**: confirmed | uncertain
- **What**: ...
- **Evidence**: ...
- **Behavior safety**: ...
- **Why**: ...
- **Fix**: ...
- **Validation**: ...

## Decision Queue

List `decision-only` findings here, not in Actionable Findings. Include the
product question that must be answered before any change.

## Partial Coverage Notes

Mention failed subagents, skipped domains, generated-code exclusions, or bug
findings that belong in a separate audit.
```

The root agent returns the absolute path to `REPORT_PATH` plus a 3-line
summary:

- total actionable findings
- HIGH-severity actionable count
- top 3 categories by total actionable findings

---

## Final Output

After final synthesis completes:

1. Print the absolute path of `REPORT_PATH`.
2. Print the 3-line summary.
3. State that no source files were modified.

## Notes

- This is a report-only skill. Do not edit source, config, tests, docs, or
  generated files while running it.
- Prefer nested delegation over flat orchestration: root owns project synthesis,
  domain coordinators own domain fan-out, verifiers own confirmation and dedupe.
- Launch sibling agents in parallel whenever their work is independent.
- If a subagent fails, its parent records which role failed and continues with
  available findings. The final report must note partial coverage.
- Each subagent prompt must be self-contained and include the Non-Functional
  Contract, Evidence Filter, concrete `<cwd>`, `<PROJECT_NAME>`, and assigned
  `<domain.*>` values. Include the Module Split Filter for module boundary
  specialists and verifiers.
- Use the strongest available model and highest practical reasoning effort for
  coordinator, specialist, and verifier agents.
- If the user later asks to apply findings, apply only confirmed actionable
  items, preserve behavior, run the named validation checks, and leave
  `decision-only` or `uncertain` items untouched until confirmed by the user.
