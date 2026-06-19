---
name: agents-md-align
description: "Update or create the repository root AGENTS.md so it accurately reflects the current project. Use when the user asks to update AGENTS.md, align AGENTS.md with the project, refresh repository agent instructions, create project-specific Codex guidance, audit stale AGENTS.md guidance, or make Codex instructions match repo layout, commands, conventions, and validation practices."
---

# AGENTS.md Align

Update the repository root `AGENTS.md` from verified project evidence. This
skill uses a subagent fan-out workflow similar to an audit, but the only
permitted source mutation is the root `AGENTS.md`.

Treat `AGENTS.md` as durable project guidance for future Codex runs: concise,
repo-specific, command-oriented, and grounded in current files.

## Invocation

- `$agents-md-align`, `/agents-md-align`, or "update AGENTS.md" -> target the
  repository root `AGENTS.md`.
- If no repository root is found, target `./AGENTS.md` in the current working
  directory and mention this in the final response.
- Create `AGENTS.md` if it is missing or empty.
- Update only the root `AGENTS.md`. Detect nested `AGENTS.md` files as context,
  but do not edit them in v1.

If subagent tools are unavailable or policy blocks spawning subagents, stop
before editing and tell the user that this skill requires explicit parallel
subagent authorization. Do not silently fall back to a single-agent rewrite.

## Mutation Contract

Subagents are read-only. The root agent alone edits the root `AGENTS.md` after
verification.

Do not modify source files, configs, tests, generated files, nested
`AGENTS.md` files, README files, or package metadata. If another file appears
necessary, report that as a follow-up instead of editing it.

Preserve existing valid human guidance. Remove or replace only guidance that is
stale, generic, contradicted by verified repo evidence, scoped too broadly for
the root file, or not useful to future agents.

## Quality Bar

A strong root `AGENTS.md` is:

- **Durable**: contains reusable repo guidance, not one-off task instructions.
- **Project-specific**: every substantive rule is backed by local evidence:
  manifests, CI, README/docs, scripts, config, tests, source layout, or repeated
  source patterns.
- **Practical**: covers layout, setup/build/test/lint commands, coding
  conventions, validation expectations, PR/review norms when evidenced, and
  repo-specific do-not rules.
- **Concise**: prefer roughly 80-160 lines; keep it under 12 KiB unless
  preserving critical existing guidance requires more.
- **Layer-aware**: root guidance covers the whole repo. Highly local rules
  belong in nested `AGENTS.md` files and should be mentioned only as scope
  boundaries.
- **Agent-useful**: commands include exact invocation and working directory when
  non-obvious; conventions say what to do, not vague philosophy.
- **Clean**: no secrets, generic best-practice essays, stale README copy,
  long architecture tours, or restatements of base Codex behavior.

Reject content whose natural evidence is "this is good practice" rather than
"this project does this here."

## Evidence Filter

Accept a proposed instruction only when all are true:

1. **Local evidence** - cite the file(s) proving the instruction.
2. **Current truth** - the referenced command, path, convention, or constraint
   exists now.
3. **Agent value** - future Codex runs need it to work safely or efficiently in
   this repo.
4. **Right scope** - it belongs in the root `AGENTS.md`, not a nested file,
   task prompt, global preference, project config, hook, skill, or README.
5. **Low ambiguity** - the wording can be followed without product or team
   judgment.

Reject:

- generic language such as "write clean code", "follow best practices", or
  "add tests where appropriate" unless the repo has a concrete local standard
- commands that are not present in manifests, CI, Makefiles, scripts, docs, or
  package metadata
- conventions inferred from one isolated example
- stale instructions contradicted by current files
- workflow preferences that belong in user-global `~/.codex/AGENTS.md`
- durable configuration that belongs in `.codex/config.toml`
- mechanical enforcement better handled by hooks or lint rules
- lengthy architecture explanation better left in README or docs
- secrets, tokens, private credentials, or operational runbooks that should not
  be loaded automatically into agent context

## Pipeline

### 1. Root Discovery

The root agent identifies the repo root, existing root `AGENTS.md`, nested
instruction files, and logical project domains. Do not write discovery
artifacts unless needed to manage oversized output.

Inspect:

- manifests and package files: `package.json`, `pnpm-workspace.yaml`,
  `pyproject.toml`, `Cargo.toml`, `go.mod`, `Gemfile`, `Makefile`, etc.
- CI and automation: `.github/workflows`, build scripts, deploy config,
  pre-commit config, lint/type/test config
- docs that describe real setup or contribution flow: README, CONTRIBUTING,
  docs, Storybook/config docs
- production source layout, shared libraries, tests, fixtures, generated output
  markers, and important config directories
- existing root and nested `AGENTS.md` / `AGENTS.override.md` files

Ignore vendored dependencies, lockfiles, generated output, build artifacts,
`.git`, `node_modules`, `.next`, `dist`, `build`, `coverage`, and caches except
when they prove a generated-output exclusion rule.

Build in memory:

- `repo_root`
- `target_path`
- `project_type`
- `domains`: logical areas with paths and purpose
- `existing_guidance`: valid, stale, and uncertain root `AGENTS.md` content
- `nested_guidance`: nested files and their scope boundaries
- `cross_cutting_evidence`: commands, conventions, generated paths, validation
  expectations, and repo-specific hazards

### 2. Fan-Out

Launch one domain coordinator subagent per logical domain, in parallel when
possible. Each coordinator owns exactly one domain and is read-only.

Also launch cross-cutting specialist subagents in parallel for:

- command and environment discovery
- existing `AGENTS.md` drift
- CI, lint, typecheck, test, and validation alignment
- generated, vendored, and do-not-edit paths
- commit, PR, or review conventions when evidence exists

If nested subagent depth is available, each domain coordinator launches its own
specialists. If only direct children are allowed, the root agent launches the
same specialist roles directly and merges their packets.

Each subagent prompt must be self-contained and include:

- concrete `cwd`, `repo_root`, target `AGENTS.md`, and assigned paths
- the Mutation Contract
- the Quality Bar
- the Evidence Filter
- an instruction to return only verified, citation-backed guidance candidates

### 3. Specialist Packets

Specialists return markdown packets in this exact shape:

```markdown
# AGENTS.md Evidence Packet: <role-or-domain>

## Keep
- **Instruction**: exact guidance to preserve or include
- **Section**: suggested AGENTS.md section
- **Evidence**: `path:line` references or command/config references
- **Scope**: root | nested-only | uncertain
- **Why agents need it**: one sentence

## Remove or Replace
- **Existing text**: short quote or section name
- **Evidence**: why it is stale, generic, contradicted, or wrong-scope
- **Replacement**: exact replacement text, or `delete`

## Uncertain
- **Question**: what could not be verified
- **Evidence checked**: files/searches inspected
```

Subagents must not draft the full `AGENTS.md`; they gather evidence and
instruction candidates only.

### 4. Verify + Merge

The root agent verifies every candidate before editing:

1. Open the cited source files and surrounding context.
2. Confirm commands exist and determine the correct working directory.
3. Confirm conventions appear in docs/config or at least two meaningful local
   examples.
4. Check nested `AGENTS.md` scope so root guidance does not override local
   instructions.
5. Drop candidates that fail the Evidence Filter.
6. Deduplicate overlapping candidates and prefer the shortest clear wording.
7. Preserve existing correct guidance when only minor wording changes are
   needed.

Keep a brief in-memory change map of sections to add, update, preserve, delete,
or leave uncertain.

### 5. Write AGENTS.md

Use this section order unless the existing file has a strong, still-valid local
structure:

```markdown
# Repository Guidelines

## Project Structure & Module Organization

## Build, Test, and Development Commands

## Coding Style & Naming Conventions

## Testing Guidelines

## Commit & Pull Request Guidelines

## Agent-Specific Instructions
```

Omit `Commit & Pull Request Guidelines` when no repo-specific evidence exists.
Add a short repository-specific section only when the standard sections cannot
cleanly hold a verified instruction.

Write in short paragraphs and flat bullets. Use exact commands in backticks.
Prefer examples with concrete paths. Do not include citations in the final
`AGENTS.md`; citations belong in the final response summary or intermediate
verification notes.

When updating an existing file:

- keep correct instructions even if wording changes slightly for clarity
- delete empty or generic sections
- replace stale commands with verified current commands
- move highly local guidance out of the root file by replacing it with a short
  note that nested `AGENTS.md` files carry local rules, but do not edit those
  nested files

### 6. Final Response

Return:

1. Whether root `AGENTS.md` was created or updated.
2. The absolute path of the edited file.
3. A concise summary of major section changes.
4. Key validation performed, including any commands run.
5. Any uncertain items or recommended follow-ups.
6. State that no files other than root `AGENTS.md` were modified.

## Notes

- `AGENTS.md` is loaded automatically before Codex work; closer nested files
  override broader root guidance because they appear later in the instruction
  chain.
- Empty instruction files are ignored. Very large combined guidance may be
  truncated by project instruction byte limits, so brevity matters.
- If the user asks to update nested guidance in a later task, treat that as a
  separate scoped update and repeat the same evidence filter for each nested
  target.
