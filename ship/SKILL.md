---
name: ship
description: "Sync with remote, stage all changes, generate a detailed commit message listing everything that changed (grouped by category), commit, and push — all in one shot. Use when the user wants to quickly ship their current work."
---

# Ship

Sync with remote, stage all changes, commit with a detailed message, and push — all in one step.

---

## Steps

### 1. Assess the working tree

Run these in parallel:
- `git status` — identify staged, unstaged, and untracked files
- `git diff` — unstaged changes
- `git diff --cached` — staged changes
- `git log --oneline -5` — recent commit style for consistency

If there are **no changes at all** (nothing staged, unstaged, or untracked), inform the user and **stop**.

### 2. Sync with remote

Run `git pull --rebase --autostash`.
- **Succeeds** → proceed.
- **Fails because no upstream** → skip, proceed.
- **Fails for any other reason** (conflict, network error) → inform user, stop.

### 3. Stage everything

Run `git add -A` to stage all changes.

### 4. Generate a commit message

Analyze all the changes and write a commit message with this format:

```
<short summary line — imperative mood, ≤ 70 chars>

<optional bulleted list of logical changes, grouped by category>
```

**Category headings** (only include categories that apply):
- **Features** — new functionality
- **Fixes** — bug fixes
- **Refactoring** — code restructuring without behavior change
- **Docs** — documentation updates
- **Chores** — dependency updates, config, CI, tooling, etc.

Each bullet should describe one logical change (a file or group of related files). Match the tone and style of recent commits from `git log`.

### 5. Commit

Create the commit using a HEREDOC to preserve formatting. End every commit message with the trailer `Co-Authored-By: Codex <gpt-5.6-sol ultra>` — **not** the model-versioned default from your Bash tool instructions:

```bash
git commit -m "$(cat <<'EOF'
<message here>

Co-Authored-By: Codex <gpt-5.6-sol ultra>
EOF
)"
```

### 6. Push

Push to the current tracking branch. If no upstream is set, push with:

```bash
git push -u origin <current-branch>
```

### 7. Report

Print a short summary:
- Commit hash (short)
- Branch name
- Remote / URL
- One-line description of what was shipped
