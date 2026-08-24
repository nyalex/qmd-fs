# Obsidian Vault Skill for Claude

Give Claude the ability to search, read, create, and organize notes in your
[Obsidian](https://obsidian.md) vault — from **Claude Desktop** and **Claude Code**.

## Why

Knowledge you build with Claude gets stranded in the conversation that produced
it. You end up with twenty open chats and no idea which one had the deployment
steps that finally worked. Chat history isn't searchable in any useful way and
you can't build on it, so you end up solving the same problem twice.

This makes your vault the memory instead. Sessions become disposable; the notes
persist. You pull a note into context, work from it, then write what you learned
back into it — so the next session starts where the last one ended, in a
different chat, days later.

Obsidian suits this precisely because it's boring: plain Markdown on your own
disk. Edit a note by hand and that edit is what Claude reads next time. No export
step, no proprietary store, nothing that stops being readable if you stop using
any of these tools.

### Example: picking up where you left off

> **You:** pull in my note on the homelab Docker setup

Claude finds it by meaning rather than filename and loads it into context. You
carry on with everything you'd already worked out, without reconstructing it.

### Example: closing the loop

> **You:** add what we just figured out about the reverse proxy to that note

The note is updated in place, frontmatter and structure intact. Next month, in a
different chat, it's there.

## What's involved

Two pieces work together:

- [**qmd**](https://github.com/tobi/qmd) — a local search engine that indexes your
  vault and provides keyword, semantic, and hybrid search
- [**@modelcontextprotocol/server-filesystem**](https://www.npmjs.com/package/@modelcontextprotocol/server-filesystem)
  — a filesystem MCP server scoped to your vault, used for writes

Both servers run on your machine, and the filesystem server can only reach the
vault directory you point it at. Note content that Claude reads becomes part of
the conversation, the same as anything else you send to a chat.

## What Claude can do with this

- **Search semantically** — "find anything about meal prep" matches notes about
  "weekly food planning" or "batch cooking", not just the literal words
- **Read notes** — pull any note into context by name, topic, or docid
- **Create notes** — with correct frontmatter, in the right folder, matching your
  vault's naming convention
- **Use your templates** — resolve `{{date}}` / `{{title}}` placeholders and write the
  result where that note type belongs
- **Understand Obsidian syntax** — wikilinks, embeds, callouts, tags, Dataview blocks

## Prerequisites

- [Node.js](https://nodejs.org) **v22**. Node 25 is not supported — qmd's
  `better-sqlite3` dependency lacks prebuilds for it.
- **qmd 2.0 or later.** Earlier versions exposed different MCP tools and this skill
  will not work against them.
- [Claude Desktop](https://claude.ai/download) and/or
  [Claude Code](https://docs.anthropic.com/en/docs/claude-code)

## Installation

### 1. Install qmd

```bash
npm install -g @tobilu/qmd
qmd --version    # confirm 2.x
```

### 2. Add your vault as collections

**Add one collection per top-level folder, not one for the whole vault.** Collections
are how you say "save this to projects" or restrict a search to your dev notes. A
single vault-wide collection makes all of that impossible.

```bash
VAULT="/absolute/path/to/your/vault"

qmd collection add "$VAULT/00 Inbox"   --name inbox
qmd collection add "$VAULT/10 Projects" --name projects
qmd collection add "$VAULT/30 Dev"      --name dev
qmd collection add "$VAULT/40 Personal" --name personal
qmd collection add "$VAULT/Templates"   --name templates
```

Use whatever folders and names match your vault.

### 3. Describe each collection

Contexts tell Claude what lives where. They appear alongside every search result and
measurably improve which notes come back.

```bash
qmd context add qmd://inbox    "Quick captures and fleeting notes to triage later"
qmd context add qmd://projects "Active, time-bound work with a finish line"
qmd context add qmd://dev      "Technical reference, code snippets, runbooks"
qmd context add qmd://personal "Personal admin — finances, health, home"
qmd context add qmd://templates "Note templates"
```

### 4. Build the index

```bash
qmd update    # scan collections
qmd embed     # generate embeddings for semantic search
qmd status    # confirm document counts
```

`qmd status` only lists collections that contain indexed documents. A collection over
an empty folder won't appear — that's expected, not an error. `qmd collection list`
shows all of them.

### 5. Register the MCP servers

**Claude Desktop** — edit `~/Library/Application Support/Claude/claude_desktop_config.json`
(macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "qmd": {
      "command": "qmd",
      "args": ["mcp"]
    },
    "obsidian-vault": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/absolute/path/to/your/vault"
      ]
    }
  }
}
```

Restart Claude Desktop. Use an absolute path — no `~` or `$HOME`.

The filesystem server is confined to the directories you pass it. Path traversal,
absolute paths outside the root, and symlinks pointing outside are all rejected. Ask
Claude to call `list_allowed_directories` any time you want to see the live scope.

**Claude Code:**

```bash
claude mcp add qmd -s user -- qmd mcp
```

That's all you need. Claude Code has native file tools, so the filesystem server is
redundant there — skip it unless you want the same tool surface in both clients.

Then allow the tools so you aren't prompted every call, in `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "mcp__qmd__query",
      "mcp__qmd__get",
      "mcp__qmd__multi_get",
      "mcp__qmd__status"
    ]
  }
}
```

If you also registered the filesystem server in Claude Code, add the tools you want
to auto-allow — `mcp__obsidian-vault__read_text_file`, `__write_file`, `__edit_file`,
`__list_directory`, `__move_file`, `__search_files`, `__directory_tree`,
`__create_directory`, `__get_file_info`.

### 6. Install the skill

**Claude Desktop / Claude.ai:** Settings → Capabilities → enable **Code execution and
file creation**, then Customize → Skills → upload `obsidian-vault.skill` from the
[latest release](../../releases) and toggle it on.

**Claude Code:** skills load from `~/.claude/skills/` as directories:

```bash
unzip obsidian-vault.skill -d ~/.claude/skills/
```

### 7. Add the CLAUDE.md snippet

Copy `CLAUDE-snippet.md` into `~/.claude/CLAUDE.md` and replace the example values
with your own collections, naming convention, and templates. This is what lets you say
"save this to projects" and have it land in the right folder.

## Keeping the index current

qmd has no file watcher. New and edited notes are invisible to search until you
reindex, so set this up now rather than discovering it later.

In Claude Code the skill runs `qmd update` for you after writing. Claude Desktop can't
— qmd's MCP server exposes no reindex tool. Schedule it instead.

**macOS (launchd, recommended)** — survives sleep, unlike cron. Save as
`~/Library/LaunchAgents/com.you.qmd-refresh.plist` and
`launchctl load` it:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.you.qmd-refresh</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string><string>-lc</string>
    <string>qmd update &amp;&amp; qmd embed</string>
  </array>
  <key>StartInterval</key><integer>900</integer>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
```

**Linux (cron):**

```cron
*/15 * * * * /usr/local/bin/qmd update && /usr/local/bin/qmd embed
```

Either is a no-op when nothing changed — `qmd embed` only processes documents that
need it. Avoid file watchers if your vault sits in a sync folder (Dropbox, iCloud,
Synology); sync bursts will trigger overlapping runs.

## Templates

Put a template in your vault, add a row to the template mapping in your
`~/.claude/CLAUDE.md`, and Claude will use it when your request matches the trigger.

```markdown
---
tags: [meeting]
date: "{{date}}"
attendees:
---

# {{title}}

## Notes

## Action Items
- [ ]
```

Claude resolves `{{title}}`, `{{date}}`, `{{time}}` (core Templates plugin) and
`<% tp.* %>` (Templater), then writes the result to the collection named in your
mapping.

Quote placeholders inside frontmatter — bare `{{date}}` is invalid YAML, since `{`
opens a flow mapping. Claude writes the resolved date unquoted in the generated note.

## Usage

> *"Find anything in my vault about meal prep"*
> *"Pull in my note about project planning"*
> *"Create a new daily note for today"*
> *"What have I written about improving my morning routine?"*
> *"Read my weekly review template and create one for this week"*

## How it works

qmd handles reads and search; the filesystem server handles writes. They see the same
files but describe them differently, which matters in one specific way:

**qmd slugifies the paths it returns.** `Templates/Meeting Note.md` on disk comes back
as `templates/meeting-note.md`. The transformation isn't reversible, so the skill never
hands a qmd path to a filesystem tool — it lists the real directory and matches. If it
did otherwise you'd get duplicate notes rather than an error.

## Troubleshooting

**qmd finds nothing**

```bash
qmd status                 # what's indexed
qmd collection list        # every collection, including empty ones
qmd update && qmd embed    # reindex
qmd doctor                 # diagnose the install
```

**MCP servers not connecting**

```bash
qmd mcp --help
npx -y @modelcontextprotocol/server-filesystem "/absolute/path/to/your/vault"
claude mcp list
```

Check Claude Desktop logs at `~/Library/Logs/Claude/` (macOS). JSON syntax errors —
trailing commas especially — are the usual cause.

**MCP servers missing in a Claude Code project** — register with `-s user`, or they
only exist in the directory where you added them.

**Filesystem tools rejected with "unsupported dialect"** — current versions of
`@modelcontextprotocol/server-filesystem` declare draft-07 output schemas, which
clients built on the Claude Agent SDK reject before dispatch. This affects **Claude
Code and Cowork sessions**; Claude Desktop's own chat is unaffected and works
normally.

In Claude Code you don't need this server at all — use its native file tools, which
is why the setup above doesn't register it there.

Rolling back does **not** fix it. `2025.8.21` was the last release without output
schemas, but its dependencies are declared as caret ranges that now resolve to MCP
SDK 1.30 and zod 4, which `zod-to-json-schema@3` cannot process — it emits an empty
`inputSchema` and every tool is rejected for a different reason. Pinning the whole
tree works if you need it:

```json
{
  "dependencies": { "@modelcontextprotocol/server-filesystem": "2025.8.21" },
  "overrides": { "@modelcontextprotocol/sdk": "1.17.5", "zod": "3.25.76" }
}
```

`npm install` that, then point the server's `command` at
`node_modules/@modelcontextprotocol/server-filesystem/dist/index.js` instead of using
`npx`. Directory confinement is unaffected — path traversal, absolute paths outside
the root, and symlinks pointing out are all still rejected on that version.

See [claude-code#86142](https://github.com/anthropics/claude-code/issues/86142).

**Frontmatter not being read** — the opening `---` must be the first line with the
YAML immediately after it, and a closing `---` is required. A blank line after the
opening delimiter silently voids the block; Obsidian renders it without complaint and
qmd indexes it as body text.

## License

MIT — do whatever you want with it.
