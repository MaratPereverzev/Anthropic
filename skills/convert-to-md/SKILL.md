---
name: convert-to-md
description: Analyze a file provided as argument, decompose it into topics, extract everything important (architecture, core idea, key components, file references, DB schema, IMPORTANT flags), and rewrite it as a clean, well-structured Markdown file optimized for Claude consumption. Use when the user says "convert-to-md <file>", "analyze and convert <file>", or asks to break down a file's content into a Claude-friendly .md.
---

# Convert to MD

Turn an arbitrary input file into a clean, well-structured Markdown document that captures every topic discussed, surfaces every important takeaway, and is optimized for future Claude sessions to consume.

## User Input

```text
$ARGUMENTS
```

The argument is a **single file path** (absolute or repo-relative). You **MUST** resolve it before doing anything else. If the argument is empty, ambiguous, or the file does not exist, stop and ask the user which file to analyze using the platform's question tool (`AskUserQuestion` in Claude Code).

## Core Principle

The output `.md` is for a *future Claude*, not a human reader. Every section must be load-bearing — if a future agent could derive a fact from `git log`, the schema, or another already-existing doc, prefer a reference over restating it. Density over decoration: headings, bullets, and numbered lists are tools to make scanning fast, not to look pretty.

## Workflow

You **MUST** execute the workflow in this order. Do not skip phases. Do not collapse Phase 1 and Phase 2 into a single read.

### Phase 1 — Ingest

1. **Read the entire file.** Use the `Read` tool with no `offset`/`limit` first. If the file is larger than ~2000 lines, page through it sequentially — never sample.
2. **Detect the file type.** Identify which branch of the workflow applies:
   - **Plain text / prose / spec / RFC** → standard prose decomposition
   - **Source code** (`.py`, `.ts`, `.js`, `.go`, ...) → code-structure decomposition (modules, classes, functions, side effects)
   - **DB schema / migration / SQL / ORM model file** → schema decomposition (mandatory DB Schema section)
   - **Configuration** (`.yml`, `.toml`, `.json`, `.env`) → setting-by-setting decomposition with defaults and consumers
   - **Existing `.md`** → audit-and-fix mode (see Phase 4)
3. **Index file references.** While reading, collect every mention of another file path, module name, URL pointing at a repo file, or symbol that lives elsewhere. You will need these for the `@import` step.

### Phase 2 — Decompose (Topic Map)

Build a topic map of the file. Two passes:

1. **Enumerate every topic mentioned.** A topic is any distinct subject the file discusses — even a one-sentence mention counts. Aim for completeness here, not ranking.
2. **Rank topics by discussion weight.** Roughly score each topic by:
   - Number of lines / paragraphs devoted to it
   - Presence in headings, callouts, or repeated mentions
   - Whether the file *defines* the topic (high weight) vs *mentions* it in passing (low weight)

   The top-weighted topics become **top-level sections** in the output. The long-tail topics either become sub-bullets under the closest top-level section or are dropped if truly trivial.

   **DB schema exception:** If the file is a DB schema artifact (tables, columns, types, relationships, migrations), every table is top-weighted and nothing is dropped. Each table must become its own sub-section. There is no trivial row in a schema file.

**Do not** invent topics that the file does not discuss. Decomposition is extraction, not embellishment.

### Phase 3 — Important Takeouts

Independently of the topic map, scan the file a second time for these high-priority categories. Each one that applies **MUST** become its own dedicated section in the output (in this order, when present):

1. **Core Idea** — the single sentence that captures *why this file exists*. If the file does not state it explicitly, infer it from the dominant topic in Phase 2.
2. **Architecture** — how the pieces fit together. Diagrams in ASCII are welcome only if the file itself contains structural information; otherwise prose + bullets.
3. **Key Components** — named modules, classes, functions, services, or roles that the file introduces. One bullet per component, with a one-line purpose.
4. **DB Schema** *(only if the file is a DB-related artifact)*. For each table:
   - **Table name** (as a sub-heading)
   - **Attributes table** with three columns: `Column | Type | Notes` (nullable, default, index, unique)
   - **Foreign keys / Relationships** — list each FK with the format `column → referenced_table.referenced_column` and the relationship cardinality (1:1, 1:N, N:M)
   - **Indexes** — list non-PK indexes
5. **File References** — every other file the input points at. Render each as an `@<path>` reference (see Phase 4 §3) so Claude resolves them inline. Only reference `.md` files — never the original source files.
6. **IMPORTANT Flags** — any sentence in the file marked `IMPORTANT`, `WARNING`, `NOTE`, `CAUTION`, `TODO`, `FIXME`, or written in screaming caps. Quote them verbatim (do not paraphrase) under a dedicated `## Important` section, each as its own bullet.

If a category does **not** apply to this file, **omit the section entirely**. Do not write "N/A" placeholders.

### Phase 4 — Write the Markdown

1. **Output path.** Write the result next to the input file, replacing the extension with `.md`. Example: `notes/api_spec.txt` → `notes/api_spec.md`. If the input is already `.md`, overwrite in place (audit-and-fix mode). Confirm the destination with the user only if it would overwrite an unrelated existing `.md`.

2. **Document skeleton.** Use this structure as a default; drop any section that does not apply.

   ```markdown
   # <Title derived from the file>

   > <One-sentence Core Idea>

   ## Overview
   <2-4 sentence summary covering the file's purpose and scope>

   ## Architecture
   <Prose + bullets describing structure>

   ## Key Components
   - **<Name>** — <one-line purpose>
   - ...

   ## <Top topic 1 from Phase 2>
   <Decomposed content>

   ## <Top topic 2 from Phase 2>
   ...

   ## DB Schema
   ### <table_name>
   | Column | Type | Notes |
   |---|---|---|
   | ... | ... | ... |

   **Foreign keys**
   - `col → other_table.col` (1:N)

   **Indexes**
   - ...

   ## File References
   @path/to/related_file_1.md
   @path/to/related_file_2.md

   ## Important
   - > "<verbatim IMPORTANT line from source>"
   - > "<verbatim WARNING line from source>"
   ```

3. **File reference rules.** When the output needs to point at another document, use Claude's `@<path>` syntax — a standalone line containing only the `@` character followed immediately by the repo-relative path, no spaces, no prose on the same line. Rules:

   **Syntax**
   - Write `@path/to/file.md` — not `@import path/to/file.md`, not `[@file](...)`.
   - One reference per line. No surrounding text on the same line.
   - Always use repo-relative paths, never absolute machine paths.

   **Only reference `.md` files**
   - Never reference the original source file (`.txt`, `.py`, `.sql`, etc.) in the output. This skill exists specifically to produce `.md` versions of those files for Claude consumption. Once an `.md` version exists, it is the canonical reference.
   - If the related file has an `.md` counterpart (same path, `.md` extension), reference the `.md` version exclusively.
   - If no `.md` counterpart exists yet, omit the reference entirely and note it in `## Important` as "needs convert-to-md".
   - If the related file is external (URL, third-party docs), use a plain Markdown link instead — `@<path>` is reserved for repo-local `.md` files.

   **Prevent cyclic references**
   - Before writing any `@<path>` reference, check whether the target file already references back to the current output file (directly or through a chain). If it does, omit the reference and note it in `## Important` as "omitted — cyclic reference with `<other_file.md>`".
   - Two files may describe related concepts without referencing each other; that is always safe. Only bidirectional (A→B and B→A) or chain cycles (A→B→C→A) need to be broken.
   - When a cycle must be broken, prefer keeping the reference in the file that is lower in the conceptual hierarchy (e.g., a detail file referencing a summary file is fine; the summary file should not reference back).

   **Verify before writing**
   - Before emitting any `@<path>`, confirm the `.md` file exists with a `Read` or `ls`. If it does not exist, omit and note in `## Important`.

4. **Formatting discipline.**
   - **Headings** — exactly one `#` (title), then `##` for top-level sections, `###` for sub-sections. Do not skip levels.
   - **Bullets vs. numbered lists** — use numbered lists only when *order matters* (workflows, ranked priorities, sequential steps). Otherwise use bullets.
   - **Tables** — use for any data that has two-or-more parallel attributes per row (schemas, config matrices, comparison grids).
   - **Code blocks** — fenced with the correct language hint. Keep blocks short; if you would inline more than ~30 lines, prefer an `@import` to the source file.
   - **Quotes** — use `> ` blockquotes for verbatim text from the source (especially IMPORTANT flags).
   - **No emojis.** No decorative unicode. (See user memory: [[feedback_comment_style]] — applies to generated docs too.)

5. **Density check before writing.** Read your draft mentally as a future Claude. For each line, ask: "would removing this line lose information a future agent needs?" If no, delete it. Aim for the shortest document that still covers every Phase 2 topic and every Phase 3 takeout.

   **DB schema exception:** Skip the density filter entirely for schema content. Every column, every FK, every type annotation, every table — regardless of how "obvious" it seems — must appear in the output. A future agent querying the schema cannot infer a missing column from context.

### Phase 5 — Verify

After writing the file:

1. Re-read the output with the `Read` tool.
2. Confirm every Phase 2 top topic has a heading.
3. Confirm every Phase 3 category that applied has its dedicated section.
4. Confirm every `@<path>` reference points to an `.md` file that exists. Flag any that do not.
5. Confirm no two files in the output's reference graph form a cycle.
6. Report to the user:
   - Output path
   - Count of topics decomposed
   - Count of `@<path>` references emitted
   - Count of IMPORTANT flags captured
   - Any references omitted (with reason: missing `.md`, cyclic, or external)

## Audit-and-Fix Mode (input is already `.md`)

When the input file is `.md`, you are *fixing* an existing document rather than converting one. Apply the same Phase 1–3 analysis, then:

1. **Diff mentally** between the existing structure and the structure Phases 2–3 would produce.
2. **Preserve** content that already meets the standard — do not rewrite for the sake of rewriting.
3. **Fix** these specifically:
   - Missing `## Core Idea` / `## Overview` at the top
   - Headings that skip levels (`#` directly to `###`)
   - File mentions that should be `@<path>` references
   - References to non-`.md` files — replace with the `.md` counterpart, or omit and note in `## Important` if no `.md` exists
   - Cyclic references — detect and break cycles, keeping the reference in the lower-hierarchy file
   - IMPORTANT flags buried in prose instead of in a dedicated section
   - DB schema described in prose instead of in tables
   - Walls of text that should be bulleted or numbered
   - Stale `@<path>` references (verify each target exists)
4. **Overwrite the file in place** with `Edit` (preferred, smaller diff) or `Write` (if the rewrite is extensive).
5. In the report to the user, list the categories of fixes applied — not a full diff.

## Anti-Patterns to Avoid

- **Do not** summarize without decomposing. A single "Summary" section is the failure mode of this skill.
- **Do not** invent architecture or components the source file does not describe.
- **Do not** paraphrase IMPORTANT flags — quote them verbatim, even if the wording is awkward.
- **Do not** use `@import path` syntax — the correct syntax is `@path` with no space and no `import` keyword.
- **Do not** reference non-`.md` files. The whole point of this skill is to produce `.md` files that Claude can consume directly. Referencing the original `.txt`, `.py`, `.sql`, etc. defeats the purpose.
- **Do not** create cyclic references. If A references B and B already references A, break the cycle by removing the reference from the lower-priority document.
- **Do not** delete the original file. Write the `.md` alongside it (unless input was already `.md`, in which case overwrite is correct).
- **Do not** ask the user for confirmation between phases — execute the full workflow and report at the end. Only stop early if the input path is unresolvable.

## Notes

- The output `.md` is a *Claude-facing artifact*. Optimize for fast scanning and unambiguous extraction, not human prose readability.
- When the source file is very long (>1000 lines), the topic map in Phase 2 becomes the most important step — get the ranking right or the output will bury the lede.
- For schema files, the DB Schema section is non-negotiable and must use tables, not prose. A future Claude needs to be able to find a column type in O(1) glance time.
