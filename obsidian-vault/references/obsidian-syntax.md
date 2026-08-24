# Obsidian Markdown Syntax Reference

A reference for Obsidian-specific Markdown extensions beyond standard CommonMark.

## Table of Contents
1. Frontmatter
2. Wikilinks
3. Embeds
4. Tags
5. Callouts
6. Comments
7. Footnotes
8. Math
9. Dataview
10. Template placeholders

---

## 1. Frontmatter

YAML metadata at the top of a note, delimited by `---`:

```yaml
---
title: Note Title
date: 2026-03-06
tags:
  - project
  - meeting
aliases:
  - alternate name
cssclasses:
  - wide-page
publish: true
---
```

Common fields: `title`, `date`, `tags`, `aliases`, `cssclasses`, `publish`, `description`. Users often add custom fields for Dataview queries.

When creating frontmatter, always use ISO 8601 dates (`YYYY-MM-DD`). Tags in frontmatter don't need the `#` prefix.

**The delimiters are strict.** The opening `---` must be the very first line, the YAML
must begin on the line immediately after it, and a closing `---` is required:

```markdown
---            <- first line of the file
tags: [meeting]
---            <- required
```

A blank line after the opening `---`, or a missing closing `---`, silently voids the
block. Obsidian stops showing it as properties and qmd indexes it as body text — often
mistaking the collapsed YAML for the note's title. It renders without error, so this is
easy to miss; check with a plain-text view rather than Live Preview.

## 2. Wikilinks

Internal links to other notes:

```markdown
[[Note Name]]                    → link to note
[[Note Name|Display Text]]      → link with custom display text
[[Note Name#Heading]]           → link to specific heading
[[Note Name#^block-id]]         → link to specific block
[[Note Name#Heading|Display]]   → heading link with display text
```

When creating content that references other vault notes, always use wikilinks rather than standard Markdown links.

## 3. Embeds

Embed content from other notes or files inline:

```markdown
![[Note Name]]                   → embed entire note
![[Note Name#Heading]]          → embed specific section
![[image.png]]                  → embed image
![[image.png|400]]              → embed image with width
![[document.pdf]]               → embed PDF
![[audio.mp3]]                  → embed audio
```

## 4. Tags

Tags can appear inline or in frontmatter:

```markdown
#tag
#nested/tag
#project/active
```

In frontmatter, tags are a YAML list without the `#`:
```yaml
tags:
  - tag
  - nested/tag
```

## 5. Callouts

Styled admonition blocks:

```markdown
> [!note] Optional Title
> Content of the callout

> [!warning]
> This is a warning

> [!tip] Pro Tip
> Helpful information here

> [!example]- Collapsible (collapsed by default)
> This content is hidden until expanded

> [!example]+ Collapsible (expanded by default)
> This content is visible but can be collapsed
```

Common types: `note`, `abstract`, `info`, `tip`, `success`, `question`, `warning`, `failure`, `danger`, `bug`, `example`, `quote`.

## 6. Comments

Content hidden from preview:

```markdown
%% This is a comment and won't render %%

%%
Multi-line
comment
%%
```

## 7. Footnotes

```markdown
Here is a statement[^1] with a footnote.

[^1]: This is the footnote content.

Inline footnotes are also supported^[This is an inline footnote].
```

## 8. Math (LaTeX)

```markdown
Inline: $E = mc^2$

Block:
$$
\frac{-b \pm \sqrt{b^2 - 4ac}}{2a}
$$
```

## 9. Dataview

Plugin that enables database-like queries. Appears in code blocks:

````markdown
```dataview
TABLE date, tags
FROM "Projects"
WHERE status = "active"
SORT date DESC
```

```dataview
LIST
FROM #meeting
WHERE date >= date(2026-01-01)
```
````

Don't try to execute Dataview queries. If you encounter them, note what data they're querying so you can help the user find that information manually.

## 10. Template placeholders

Two different plugins provide placeholders in template files. A vault may use either,
or both. When creating a note *from* a template, replace placeholders with resolved
values. When merely reading or copying a template file, leave them as-is.

### Core Templates plugin (built in)

Uses double braces:

```markdown
{{title}}              -> title of the note being created
{{date}}               -> today's date, default format YYYY-MM-DD
{{time}}               -> current time, default format HH:mm
{{date:YYYY-MM-DD}}    -> custom format, Moment.js tokens
{{time:HH:mm:ss}}      -> custom format, Moment.js tokens
```

Default formats are configurable under **Settings -> Core plugins -> Templates**.

**YAML caveat.** In frontmatter, `{` opens a flow mapping, so a bare placeholder is
invalid YAML:

```yaml
date: {{date}}      # invalid
date: "{{date}}"    # valid
```

Quote placeholders inside frontmatter so the template file itself stays parseable.
When you generate a note from it, write the resolved value **unquoted** so it matches
normal note frontmatter:

```yaml
date: 2026-03-06
```

### Templater plugin (community)

Uses `<% %>`:

```markdown
<% tp.date.now("YYYY-MM-DD") %>
<% tp.file.title %>
<% tp.file.cursor() %>
```

Preserve Templater syntax as-is when reading a template. Resolve it when creating a
note from one — `<% tp.date.now("YYYY-MM-DD") %>` becomes today's actual date.

Templater expressions can execute arbitrary JavaScript. Resolve only simple date and
title expressions; if a template contains logic you cannot evaluate confidently, copy
it verbatim and tell the user which expressions you left unresolved.
