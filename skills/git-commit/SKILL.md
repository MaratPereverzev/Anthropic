---
name: git-commit
description: Create one or more git commits from the current working tree by grouping changes into logical, dependency-ordered commits. Use when the user says "commit", "commit this", "save my changes", "create a commit", or wants to commit staged or unstaged work. Produces tag-grouped commit messages (Add / Refactor / Hotfix / ...) and orders commits so no commit depends on a later one.
---

# Git Commit

Split the current working tree changes into one or more logically grouped, dependency-ordered commits. Each commit gets a tag-grouped message (Add / Refactor / Hotfix / ...).

## Workflow

### Step 1: Gather context

Run these commands to understand the current state.

```bash
git status
git diff HEAD
git diff --staged
git branch --show-current
git log --oneline -10
git rev-parse --abbrev-ref origin/HEAD
```

The last command returns the remote default branch (e.g., `origin/main`). Strip the `origin/` prefix to get the branch name. If the command fails or returns a bare `HEAD`, try:

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

If both fail, fall back to `main`.

If the `git status` result from this step shows a clean working tree (no staged, modified, or untracked files), report that there is nothing to commit and stop.

Run `git branch --show-current`. If it returns an empty result, the repository is in detached HEAD state. Explain that a branch is required before committing if the user wants this work attached to a branch. Ask whether to create a feature branch now. Use the platform's blocking question tool (`AskUserQuestion` in Claude Code, `request_user_input` in Codex, `ask_user` in Gemini). If no question tool is available, present the options and wait for the user's reply before proceeding.

- If the user chooses to create a branch, derive the name from the change content, create it with `git checkout -b <branch-name>`, then run `git branch --show-current` again and use that result as the current branch name for the rest of the workflow.
- If the user declines, continue with the detached HEAD commit.

Run `git branch --show-current`. If it returns `main`, `master`, or the resolved default branch, warn the user and ask whether to continue committing here or create a feature branch first. Use the platform's blocking question tool as above. If the user chooses to create a branch, derive the name, create it with `git checkout -b <branch-name>`, and use that branch for the rest of the workflow.

### Step 2: Inventory all changes

Include **both** staged and unstaged files, plus untracked files. For each changed file, note:

- The path
- The change kind (added / modified / deleted / renamed)
- A one-line description of *what* changed (from the diff, not the filename)
- A tag classification (see Step 4)

Skip files that look like secrets (`.env`, `credentials.*`, `*.pem`, `*.key`) — never stage them automatically. If such a file appears in the working tree, list it back to the user and ask whether to commit it before proceeding.

### Step 3: Group changes into commits

Cluster files into **logical groups**. Each group becomes one commit. Heuristics:

- Files implementing the same feature/fix belong together (source + tests + migrations + docs for that feature).
- Pure refactors that touch many files but don't change behavior can be their own commit.
- Hotfixes, dependency bumps, config changes, and docs-only changes are often their own commit.
- A single change of similar tag across many files can stay as one commit — don't over-slice.
- Aim for **1–4 groups**. If you find yourself producing more, reconsider whether the splits add value.

Group at the **file level only**. Do not use `git add -p` or try to split hunks within a single file. If one file legitimately belongs in two different conceptual groups, place it in the group that owns the *dominant* change and mention the secondary concern in that commit's bullets.

### Step 4: Tag classification

For each file (or each clear change inside a multi-purpose file), assign one of the following tags. These map directly to the section headings in the commit message body:

| Tag | When to use |
|---|---|
| `Add` | New files, new features, new endpoints, new modules, new tests for new behavior |
| `Update` | Enhancement to an existing feature (new capability, expanded behavior) |
| `Refactor` | Behavior-preserving restructuring, renames, code moves, prompt rewrites that don't change semantics |
| `Hotfix` | Urgent bug fix that addresses incorrect behavior in production code |
| `Fix` | Non-urgent bug fix |
| `Remove` | Deleted files, removed features, retired code paths |
| `Docs` | Documentation only (`*.md`, comments, docstrings) |
| `Test` | Test-only changes that don't accompany a feature change |
| `Chore` | Dependency bumps, build/CI config, formatting, generated files |
| `Perf` | Performance improvements with no behavior change |
| `Security` | Security-related fixes or hardening |

If a single commit group has changes belonging to multiple tags (common), they will all appear as separate sections in that commit's body — see Step 6.

### Step 5: Order commits by dependency

Commits must apply in an order such that **no commit depends on a later one**. A commit "depends" on another if the code it introduces calls into, imports from, or otherwise requires code that only exists after the other commit is applied.

Procedure:

1. Build a dependency graph between groups. Edge `A → B` means "group A depends on something introduced in group B" (so B must commit first).
2. Topologically sort the groups. The sorted order is the commit order.
3. **Cycle resolution**: if two groups depend on each other, do not split them as-is. Move the smallest set of files between the groups to break the cycle, preferring to move files that more naturally belong with their new group's tag. If breaking the cycle requires moving the majority of one group's files, just merge the two groups into one commit instead.
4. If you had to merge or rebalance, recompute the topological order before committing.

Beyond hard dependencies, prefer this conventional order when no dependency forces otherwise: `Remove` / `Refactor` first → `Hotfix` / `Fix` → `Add` / `Update` → `Test` / `Docs` / `Chore` last. This keeps mechanical changes at the bottom of the branch history.

Before the first `git add`, present the user with the plan: ordered list of groups, the files in each, and the tag breakdown. Use the platform's blocking question tool to ask whether to proceed, edit the plan, or collapse into a single commit. Wait for the user's reply.

### Step 5.5: Pre-flatten formatters (run linters / formatters once up-front)

If the repo runs auto-formatters on commit, running them per-group causes every commit to fail-rewrite-restage-retry. Run them **once** against the full planned file set instead, so the per-group commits in Step 6 land cleanly. Walk the fallback chain below in order — stop at the first tier that produces a usable tool.

#### Tier A — pre-commit framework

Detect:

```bash
test -f .pre-commit-config.yaml && echo "pre-commit configured"
test -x .git/hooks/pre-commit && echo "pre-commit installed"
command -v pre-commit && echo "pre-commit available"
```

If present, run the flatten pass:

1. **Stage everything in the plan** with one `git add` listing every file across all groups. The staging set is only there to give pre-commit a target.
2. **Run the hooks** without committing:

   ```bash
   pre-commit run --files <file1> <file2> ...
   ```

   Hooks may modify files in place. First-pass exit code 1 from formatters is expected — not a failure.
3. **Re-stage** any files the hooks touched (`git add <same list>`), then run `pre-commit run --files ...` again to confirm clean. The second run should pass.
4. If the second run still fails with a non-formatter error (e.g. `check-added-large-files` rejecting a binary, `check-yaml` on a malformed file, a linter that does not auto-fix), stop and surface the failure to the user — it's a real issue, not a flatten loop. Ask whether to drop the offending file, edit it, or abort.
5. **Unstage everything** with `git reset HEAD` so each group can stage fresh in Step 6. The formatted versions remain in the working tree.

If the pre-commit framework is configured but `.git/hooks/pre-commit` is missing (someone forgot `pre-commit install`), fall through to Tier B instead of running with no protection.

#### Tier B — project-native linters / formatters

If pre-commit isn't usable, look for standalone formatter/linter configuration in the repo. Detect each by config file or by the tool's section in a manifest:

| Tool | Detection | Invocation (against the planned file list) |
|---|---|---|
| `ruff format` + `ruff check --fix` | `ruff.toml`, `pyproject.toml [tool.ruff]`, `.ruff.toml` | `ruff format <files>` then `ruff check --fix <files>` |
| `black` | `pyproject.toml [tool.black]`, `.black` | `black <files>` |
| `isort` | `.isort.cfg`, `pyproject.toml [tool.isort]` | `isort <files>` |
| `prettier` | `.prettierrc*`, `prettier` key in `package.json` | `npx prettier --write <files>` |
| `eslint` (with `--fix`) | `.eslintrc*`, `eslint.config.*`, `eslint` key in `package.json` | `npx eslint --fix <files>` |
| `biome` | `biome.json`, `biome.jsonc` | `npx biome format --write <files>` then `npx biome lint --apply <files>` |
| `gofmt` / `goimports` | `go.mod` present | `gofmt -w <files>` (and `goimports -w <files>` if installed) |
| `rustfmt` | `Cargo.toml` and `.rustfmt.toml` or default | `rustfmt <files>` |

Procedure:

1. Filter the planned file list per tool (don't pass `.py` files to prettier, don't pass `.ts` files to black, etc.).
2. Run formatters first (rewrite-in-place), then auto-fixing linters.
3. If a tool exits non-zero with rewrites, re-run once to verify a stable fixed-point. If it still rewrites or still errors, surface the output to the user — it's a real lint issue, not a flatten loop.
4. Do **not** stage during Tier B — the tools operate on the working tree directly. Per-group `git add` in Step 6 will pick up the formatted versions.

Verify each tool is actually installed before invoking it (`command -v ruff`, `npx --no-install prettier --version`, etc.). A config-file-without-the-binary means the project relies on CI to format — don't try to install anything; fall through to Tier C.

#### Tier C — no tooling found

If neither Tier A nor Tier B produced a runnable tool, notify the user explicitly: "No pre-commit hooks or local formatters/linters were detected. Commits will go in unformatted — CI or reviewers may flag style issues."

Use the platform's blocking question tool to ask: **proceed without formatting** or **abort and let me set something up**. Wait for the reply. Do not silently continue.

### Step 6: Commit each group

For each group, in the order determined in Step 5:

1. Stage **only** that group's files explicitly by path with `git add <file1> <file2> ...`. Never use `git add -A`, `git add .`, or `git add -p`. Verify with `git diff --staged --name-only` that only the intended files are staged.
2. Draft the commit message in the tag-grouped format below.
3. **Request human approval for this specific group before committing.** Present:
   - The group number and its position in the overall sequence (e.g. "Commit 2 of 3").
   - The exact list of files now staged for this commit.
   - The full drafted commit message verbatim.

   Use the platform's blocking question tool (`AskUserQuestion` in Claude Code, `request_user_input` in Codex, `ask_user` in Gemini) to ask the user whether to: **commit as drafted**, **edit the message**, **edit the file selection**, **skip this group**, or **abort the remaining groups**. Wait for the user's reply.
   - If the user edits the message or selection, apply the edits, re-stage if needed, re-display the updated state, and ask again. Do not commit until the user explicitly approves *this* group.
   - If the user skips the group, run `git restore --staged <files>` to unstage it and move on to the next group. A skipped group that other groups depend on means the dependent groups must also be skipped — warn the user and ask before unstaging the dependents.
   - If the user aborts, unstage everything in the current group, leave already-completed commits in place, and stop.
4. Only after explicit approval, run the commit with a heredoc to preserve formatting.

**Commit message format**:

```
<Tag>:
- <one-line description of a change>
- <one-line description of another change>

<Other Tag>:
- ...
```

Rules for the message:

- One section per tag that actually has entries. Omit any tag that has zero bullets.
- Section headings are the tag word followed by `:` (e.g., `Add:`, `Refactor:`, `Hotfix:`).
- Each bullet is a short, imperative description focused on *why* / *what value*, not a restatement of the filename.
- Order sections within a commit by significance: `Hotfix` / `Fix` first if present, then `Add`, `Update`, `Remove`, `Refactor`, `Perf`, `Security`, `Test`, `Docs`, `Chore`.
- If a commit has only one bullet under one tag, still use the tag heading + bullet form — do not collapse to a subject line.
- No trailing summary, no footer trailers (no `Co-Authored-By`, no `Generated with` lines) unless the user has explicitly asked for them.

Use a heredoc:

```bash
git commit -m "$(cat <<'EOF'
Add:
- new endpoint for X
- migration for new Y column

Refactor:
- extract Z helper out of the controller
EOF
)"
```

If a pre-commit hook fails at this point — Step 5.5 should have already flattened formatters across the whole set, so any hook failure here is a real lint/test/security issue, not a formatter rewrite. Fix the underlying issue, re-stage the affected files, and create a new commit (do not `--amend`). If a formatter does still rewrite a file (e.g. Step 5.5 was skipped because the framework wasn't installed), re-stage and retry.

### Step 7: Confirm

After all groups are committed:

- Run `git status` to verify a clean working tree (or, if anything was intentionally left unstaged, confirm what remains).
- Run `git log --oneline -<N>` where N is the number of commits created.
- Report to the user: number of commits, each commit's hash and first-line tag, and any files that were intentionally left out (e.g., suspected secrets the user declined to commit).
