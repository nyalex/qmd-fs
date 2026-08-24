---
name: obsidian-vault
description: "Use this skill whenever the user wants to interact with their Obsidian vault — reading notes, creating new notes, searching for content, or organizing files within the vault. Trigger on any mention of 'Obsidian', 'vault', 'notes', 'daily note', 'journal', 'zettelkasten', or when the user asks to 'pull in', 'grab', 'find', or 'look up' a note. Also trigger when the user asks to 'create a note', 'write a note', 'save this as a note', or 'add to my vault'. Even if the user doesn't say 'Obsidian' explicitly, trigger when they reference their personal notes, knowledge base, or second brain. Do NOT use for general Markdown file creation unrelated to the vault, or for notes in other apps like Notion or Apple Notes."
---

# Obsidian Vault Access

Read, search, create, and organize notes in the user's Obsidian vault.

Two servers work together:

- **`qmd`** — a local search engine that indexes the vault. Use it for **all** search and read operations. Tools: `query`, `get`, `multi_get`, `status`.
- **`obsidian-vault`** — a filesystem MCP server scoped to the vault directory. Use it for **all** writes. Tools: `write_file`, `edit_file`, `create_directory`, `list_directory`, `move_file`, `directory_tree`, `search_files`, `get_file_info`, `read_text_file`.

Requires **qmd 2.0 or later**. Older versions exposed `search` / `vector_search` /
`deep_search`; those were removed in qmd 1.1.0 and replaced by the single `query` tool
described below.

---

## Critical: qmd paths are not filesystem paths

qmd normalizes the paths it returns. **Both the collection segment and the filename
are slugified:**

| On disk | Returned by qmd |
|---|---|
| `Templates/Meeting Note.md` | `templates/meeting-note.md` |
| `40 Personal/Health Records.md` | `personal/health-records.md` |

The slug is **not reversible**. Given `meeting-note.md` you cannot tell whether the
file is `Meeting Note.md`, `meeting-note.md`, or `Meeting-Note.md`.

**Never pass a qmd path to a filesystem tool.** The write will not fail — it will
silently create a duplicate note beside the real one.

To act on a file found through qmd:

1. Map the collection segment to its folder (see *Collections* below)
2. `list_directory` on that folder
3. Match the slug against the real filenames — compare case-insensitively and treat
   `-` and space as equivalent
4. Use the **real** filename for the write

Reads need none of this: qmd's `get` and `multi_get` accept qmd paths directly.

Never infer the vault's naming convention from qmd paths either — they are always
slugified, so every vault looks kebab-case through qmd. Read the convention from an
actual `list_directory` listing.

---

## Collections

A collection is a named qmd index over one folder. `status` returns every collection
with its absolute path — **call it once per session and treat that as the
authoritative collection → folder map.**

```
personal  → /path/to/vault/40 Personal
projects  → /path/to/vault/10 Projects
templates → /path/to/vault/Templates
```

The user's `CLAUDE.md` may also define a collection mapping with descriptions of what
belongs where. Use `CLAUDE.md` for *intent* (which collection a new note belongs in)
and `status` for *paths* (where that collection actually lives on disk). When they
disagree, `status` wins — it reflects the live index.

Two things to know:

- **`status` omits collections with zero indexed documents.** A collection missing
  from `status` is not necessarily unconfigured; the folder may simply be empty.
  `qmd collection list` shows all of them.
- **A folder is not necessarily a collection.** Writing to a folder that no collection
  covers is legal, but the note will never be searchable. If the user asks to save
  somewhere outside every collection path, write it and say so plainly.

To restrict a search, pass collection names in the `collections` parameter.

---

## Searching

qmd exposes one search tool, `query`. It takes a list of typed sub-queries in
`searches`, not a plain string. **The first sub-query carries 2× weight — put the
strongest signal first.**

### Sub-query types

**`lex`** — BM25 keyword search. Fast, exact, no model needed. Supports:

- `term` — prefix match (`perf` matches `performance`)
- `"exact phrase"` — must appear verbatim
- `-term` / `-"phrase"` — exclude documents containing this

**`vec`** — semantic vector search. Write a natural-language question. Matches by
meaning rather than wording.

**`hyde`** — hypothetical document. Write 50–100 words that look like the answer you
expect. Often strongest for nuanced topics.

### Choosing types

| Situation | Use |
|---|---|
| You know the exact term, name, or filename | `lex` alone |
| The user described a concept, not words | `vec` alone |
| You want good recall on a real question | `lex` + `vec` |
| Broad, ambiguous, or high-stakes | `lex` + `vec` + `hyde` |
| You don't know the vault's vocabulary | one standalone natural-language query, so the server can auto-expand it |

### Other parameters

- `collections` — restrict to named collections
- `intent` — background context to disambiguate; does not search on its own.
  E.g. `query: "performance"`, `intent: "web page load times"`
- `limit` (default 10), `minScore` (0–1)
- `rerank` — LLM re-ranking, on by default. Set `false` for faster results on
  CPU-only machines or when you just need a quick existence check
- `candidateLimit` — how many candidates to rerank (default 40)

### Reading results

Results include a `score`, the qmd `file` path, a `docid`, the collection's `context`
description, and a snippet. Prioritize high scores (0.8+ strongly relevant, 0.5–0.8
moderate). Present paths and snippets to the user, then pull full content with `get`.

If a search returns nothing, say so rather than guessing. Consider whether the target
folder is covered by a collection at all.

---

## Reading notes

1. Exact filename → `get` with the qmd path
2. Topic or partial name → `query`, then `get` the best match
3. Several plausible matches → show a short list with scores and snippets and ask

`get` also takes `fromLine`, `maxLines`, and `lineNumbers` for large notes.
`multi_get` takes a glob or comma-separated list — **the glob is collection-relative
and slugified**, so `templates/*.md` matches and `Templates/*.md` returns nothing.

Notes may contain YAML frontmatter, `[[wikilinks]]`, `![[embeds]]`, `#tags`,
`> [!callout]` blocks, and Dataview code blocks. Resolve wikilinks by searching for
the linked note when the user needs that content. Never execute Dataview queries —
note what they query and help the user find it another way.

Preserve original formatting when quoting note content. Summarize long notes and
offer to show specific sections.

---

## Creating notes

### 1. Check for a template

If the user's `CLAUDE.md` defines a template mapping and the request matches a
trigger, use that template. Read it, then resolve its placeholders:

- Core Templates syntax: `{{title}}`, `{{date}}`, `{{time}}`, `{{date:YYYY-MM-DD}}`
- Templater syntax: `<% tp.date.now("YYYY-MM-DD") %>`, `<% tp.file.title %>`

Replace placeholders with resolved values and preserve everything else exactly.
Write resolved dates unquoted (`date: 2026-03-06`) even if the template quoted the
placeholder — quoting in the template is only there to keep its own YAML valid.

If no template matches, create the note from scratch.

### 2. Determine location

In order of precedence: the template's default collection → a collection the user
named → an explicit folder path → ask. Resolve collection names to folders via
`status`. Don't create new top-level folders without asking.

### 3. Determine filename

`list_directory` the target folder and match the convention you actually see there —
`Title Case With Spaces.md`, `kebab-case.md`, and `YYYY-MM-DD Title.md` are all
common. Do not assume; do not infer it from qmd paths. Always use `.md`.

### 4. Write

Include YAML frontmatter. Open with `---` on the first line, the YAML immediately
after with no blank line, and close with `---`:

```yaml
---
date: 2026-03-06
tags: []
---
```

A blank line after the opening `---`, or a missing closing `---`, silently voids the
frontmatter — Obsidian and qmd both stop treating it as metadata and index it as body
text. Check a few existing notes in the same folder for the fields that vault uses.

Use `[[wikilinks]]` for references to other vault notes, not Markdown links, and make
the link text match the target's real filename.

### 5. Confirm and reindex

Report the full path. New notes are not searchable until the index updates:

- **Claude Code:** run `qmd update`. Add `qmd embed` if the user is likely to search
  for it soon — it is slower, so batch it when creating several notes.
- **Claude Desktop:** qmd's MCP server exposes no reindex tool. Say the note was
  created but will not be searchable until the next `qmd update`. If this comes up
  often, suggest a scheduled job (see the README).

---

## Modifying notes

Only when explicitly asked.

1. Read current content with `get`
2. Resolve the real filesystem path (see *Critical* above)
3. Show what you plan to change
4. Prefer `edit_file` for targeted changes; use `write_file` only when replacing a
   note wholesale
5. Preserve all existing frontmatter fields and untouched content
6. Reindex as above

## Moving and renaming

The filesystem server's `move_file` moves bytes — it does **not** update backlinks the
way Obsidian would. After moving or renaming a note, find and fix references yourself:

1. `query` with a `lex` sub-query for `"[[Old Note Name]]"`
2. Resolve each hit to its real path
3. Update the wikilink in each file, preserving any `|display text` and `#heading`

Tell the user how many links you updated. If there are many, list the files first and
confirm before rewriting them.

---

## Daily notes

1. Check `CLAUDE.md` for a daily note mapping — template, target folder, naming pattern
2. Otherwise `list_directory` a `Daily Notes` / `Journal` / `Dailies` folder and copy
   the pattern of the notes already there
3. Resolve template placeholders, write, reindex

---

## Claude Code vs Claude Desktop

- **Claude Desktop** has no filesystem access outside MCP. The `obsidian-vault` server
  is the only write path, and reindexing must happen outside the conversation.
- **Claude Code** has native file tools and a shell. It can read and write the vault
  directly without the filesystem server, and can run `qmd update` itself. The path
  resolution rule still applies to anything found through qmd.

---

## Reference

- [references/obsidian-syntax.md](./references/obsidian-syntax.md) — Obsidian Markdown
  extensions, frontmatter conventions, template placeholder syntax, and plugin-specific
  markup. Read it when you encounter unfamiliar syntax or need to generate advanced
  Obsidian features.
