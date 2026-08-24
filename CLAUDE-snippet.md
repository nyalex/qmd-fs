## Obsidian Vault

<!-- Copy this into ~/.claude/CLAUDE.md and replace every value below with your own.
     Everything here is an example. Delete any section you don't use. -->

My Obsidian vault is at `/absolute/path/to/vault`. The `qmd` MCP server provides
search and read access; the `obsidian-vault` MCP server provides write access. Use
the obsidian-vault skill for how to read, create, and search notes.

### Collection mapping

Each row maps a qmd collection to the folder it indexes. Claude reads the folder
paths from `qmd status` at runtime — this table is here to tell Claude *what belongs
where*, so "save this to projects" lands in the right place.

| Collection | Folder path | What goes here |
|------------|-------------|----------------|
| inbox | `00 Inbox/` | Quick captures and fleeting notes to triage later |
| projects | `10 Projects/` | Active, time-bound work with a finish line |
| dev | `30 Dev/` | Technical reference, code snippets, runbooks |
| personal | `40 Personal/` | Personal admin — finances, health, home |
| people | `50 People/` | Contact notes and relationship context |
| areas | `60 Areas/` | Ongoing responsibilities with no end date |
| archive | `90 Archive/` | Finished and inactive material kept for lookup |
| templates | `Templates/` | Note templates |

Keep the descriptions specific to your own vault — they are what Claude uses to
choose a destination when you don't name one. The same text is worth setting as each
collection's qmd context (`qmd context add qmd://<name> "<description>"`), which puts
it in front of Claude on every search result.

### Naming convention

<!-- Tell Claude how your filenames look, so new notes match. Pick one. -->

Name new notes in Title Case with spaces, e.g. `Quarterly Planning.md`.

<!-- Other common conventions:
     kebab-case:    quarterly-planning.md
     date-prefixed: 2026-03-06 Quarterly Planning.md -->

### Template mapping

<!-- Delete this section if you don't use templates. -->

When I ask to create a note, check whether it matches a trigger below. If so, use
that template and default location. If nothing matches, create the note from scratch.

| Trigger | Template file | Default collection | Naming pattern | When it applies |
|---------|--------------|-------------------|----------------|-----------------|
| "meeting notes", "meeting with" | `Templates/Meeting Note.md` | projects | `YYYY-MM-DD Meeting - [topic].md` | Notes taken during or after a meeting |
| "daily note", "today's note" | `Templates/Daily Note.md` | inbox | `YYYY-MM-DD.md` | Daily journal or log entry |
| "recipe" | `Templates/Recipe Template.md` | personal | `[recipe name].md` | Cooking instructions and ingredients |
| "person", "contact" | `Templates/Person.md` | people | `[person name].md` | Notes about a specific person |

Every template file listed here must actually exist in the vault. A row pointing at a
missing file will send Claude looking for something that isn't there.
