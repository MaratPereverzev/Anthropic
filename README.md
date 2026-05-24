# anthropic-skills

Curated [Agent Skills](https://agentskills.io) for AI coding agents. Each skill is a folder with a `SKILL.md` file — the same format works across Claude Code, Cursor, Codex, Gemini CLI, and Windsurf.

```
skills/
  git-commit/
    SKILL.md
  grill-me/
    SKILL.md
  ...
```

## Install

Copy or symlink a skill folder into your agent's skills directory. The internal layout is identical everywhere: `<skills-root>/<skill-name>/SKILL.md`.

**Global** (available in every project):

```bash
ln -s /path/to/anthropic-skills/skills/git-commit ~/.claude/skills/git-commit
ln -s /path/to/anthropic-skills/skills/git-commit ~/.cursor/skills/git-commit
ln -s /path/to/anthropic-skills/skills/git-commit ~/.agents/skills/git-commit
```

**Project** (shared via git with your team):

```bash
ln -s /path/to/anthropic-skills/skills/claude-md .claude/skills/claude-md
```

### Where each agent looks

| Agent | Global | Project |
|-------|--------|---------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| OpenAI Codex | `~/.agents/skills/` | `.agents/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` |
| Windsurf | `~/.codeium/windsurf/skills/` | `.windsurf/skills/` |

Codex, Gemini, and Windsurf also discover `.agents/skills/` as a cross-agent alias.

---

## Skills

### [`git-commit`](skills/git-commit/) — global

Turns a messy working tree into one or more clean, logically grouped commits — with human approval at every step.

**Triggers:** "commit", "commit this", "save my changes", "create a commit"

**What it does:**

1. Gathers git context (`status`, diffs, branch, recent log)
2. Inventories all staged, unstaged, and untracked changes
3. Groups files into 1–4 logical commits (feature + tests + migrations together, refactors separate, etc.)
4. Tags each change (`Add`, `Refactor`, `Hotfix`, `Fix`, `Docs`, `Chore`, …)
5. Orders commits by dependency so each applies cleanly
6. Runs formatters/linters once up-front (pre-commit, ruff, prettier, eslint, …) to avoid per-commit hook loops
7. Commits each group one at a time — presenting the exact message and file list for approval before every commit

**Commit message format:**

```
Add:
- new endpoint for user preferences
- migration for preferences table

Refactor:
- extract validation helper from controller
```

**Safety rails:** warns on detached HEAD and default-branch commits, never auto-stages secrets (`.env`, keys, credentials), never uses `git add -A` or `git add -p`.

---

### [`grill-me`](skills/grill-me/) — global

Stress-tests a plan or design through relentless Q&A until every decision is resolved.

**Triggers:** "grill me", stress-test a plan, review a design before building

**What it does:**

- Walks down each branch of the design decision tree one by one
- Resolves dependencies between decisions before moving on
- Provides a recommended answer with every question
- Explores the codebase when a question can be answered from code instead of asking you

Best used before starting non-trivial work — architecture choices, feature specs, migration plans.

---

### [`convert-to-md`](skills/convert-to-md/) — global

Converts any file into a dense, Claude-optimized Markdown document for future agent sessions.

**Triggers:** `convert-to-md <file>`, "analyze and convert", "break this down into a .md"

**What it does:**

1. **Ingest** — reads the entire file; detects type (prose, source code, DB schema, config, existing `.md`)
2. **Decompose** — builds a topic map ranked by discussion weight
3. **Extract** — pulls out Core Idea, Architecture, Key Components, DB Schema (with full column tables), File References, and verbatim IMPORTANT/WARNING/TODO flags
4. **Write** — outputs `<filename>.md` next to the source with a structured skeleton
5. **Verify** — checks all topics, references, and flags are captured

**Output conventions:**

- Uses `@path/to/file.md` references (repo-relative, `.md` only, one per line)
- DB schemas get full attribute tables — no prose summaries
- IMPORTANT flags are quoted verbatim, never paraphrased
- Audit-and-fix mode when the input is already `.md`

The output is optimized for a *future Claude*, not human readability — density over decoration.

---

### [`claude-md`](skills/claude-md/) — project

Creates, updates, or audits `CLAUDE.md` — the primary onboarding document that Claude Code loads in every session.

**Triggers:** create/update/audit CLAUDE.md, set up agent context for a repo

**Modes:**

| Mode | Behavior |
|------|----------|
| `create` | Analyze the project and draft a new CLAUDE.md |
| `update` | Audit existing CLAUDE.md and improve it |
| `audit` | Report on quality without modifying the file |

**What a good CLAUDE.md contains:**

- **WHAT** — tech stack, project structure, key directories
- **WHY** — purpose, architectural decisions, component responsibilities
- **HOW** — dev commands (install, test, build), critical gotchas
- **Claude Resources** — `@` references to every `.claude/skills/`, `.claude/docs/`, and settings file with one-line descriptions

**Golden rules enforced by the skill:**

- Under 300 lines (ideally under 100)
- Only universally-applicable info — no task-specific instructions
- No style/lint rules (use deterministic tools instead)
- No code snippets (use `file:line` references)
- Progressive disclosure via `agent_docs/` for larger projects

Install this one per-repo — it reads your project's structure, `.claude/` directory, and existing docs.

---

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:

   ```yaml
   ---
   name: my-skill
   description: What it does and when to use it. Include trigger phrases.
   ---
   ```

2. Add a section to this README
3. Symlink into the agent directory you use

See [Claude Code skills docs](https://code.claude.com/docs/en/skills) for authoring guidance.
