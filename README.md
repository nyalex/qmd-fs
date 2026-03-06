# Obsidian Vault Skill for Claude

Give Claude the ability to search, read, create, and organize notes in your [Obsidian](https://obsidian.md) vault — from both **Claude Code** and **Claude.ai**.

This skill uses two tools together:
- [**qmd**](https://github.com/tobi/qmd) — A local search engine with semantic, keyword, and hybrid search for finding and reading notes
- [**@modelcontextprotocol/server-filesystem**](https://www.npmjs.com/package/@modelcontextprotocol/server-filesystem) — A filesystem MCP server for creating and organizing notes

## What Claude Can Do With This

- **Search notes semantically** — "find anything about meal prep" matches notes about "weekly food planning" or "batch cooking", not just the literal words
- **Read notes** — pull any note into context by name, topic, or docid
- **Create notes** — write new notes with proper frontmatter, respecting your vault's naming conventions and folder structure
- **Daily notes** — create daily notes following your existing template and naming pattern
- **Understand Obsidian syntax** — handles wikilinks, embeds, callouts, tags, Dataview blocks, and Templater expressions

## Prerequisites

- [Node.js](https://nodejs.org) v22 or later
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and/or [Claude Desktop](https://claude.ai/download)

## Installation

### 1. Install and configure qmd

```bash
# Install qmd
npm install -g @tobilu/qmd

# Add your vault as a collection (update the path to your vault)
qmd collection add /path/to/your/Obsidian/Vaults --name obsidian

# Optional: add context to help search understand your content
qmd context add qmd://obsidian "Personal Obsidian knowledge base"

# Generate embeddings (one-time, updates are incremental)
qmd embed
```

To keep the index current, run `qmd update && qmd embed` periodically or after adding new notes.

### 2. Register the MCP servers

You need both MCP servers — qmd for search/read and filesystem for write/organize.

**Claude Code:**

```bash
# qmd — search and read
claude mcp add qmd -- qmd mcp

# filesystem — write and organize (update the path to your vault)
claude mcp add obsidian-vault \
  --transport stdio \
  -s user \
  -- npx -y @modelcontextprotocol/server-filesystem \
  "/path/to/your/Obsidian/Vaults"
```

Verify both are registered:

```bash
claude mcp list
```

**Claude Desktop:**

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

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
        "/absolute/path/to/your/Obsidian/Vaults"
      ]
    }
  }
}
```

Then restart Claude Desktop. You should see a hammer icon (🔨) in the chat input.

### 3. Install the skill

Download `obsidian-vault.skill` from the [latest release](../../releases) and install it:

- **Claude.ai / Claude Desktop** — drag the `.skill` file into a conversation, or add it via Settings → Skills
- **Claude Code** — install per your Claude Code skill configuration

### 4. Add the CLAUDE.md snippet

Copy the contents of `CLAUDE-snippet.md` into your `~/.claude/CLAUDE.md` file (create it if it doesn't exist). Then update the vault path and collection mapping table to match your vault's folder structure and qmd collections.

The collection mapping lets you say things like "save this to projects" or "create a note in dev" and Claude will write to the correct folder.

## Usage

Once installed, just talk to Claude naturally:

> *"Find anything in my vault about meal prep"*

> *"Pull in my note about project planning"*

> *"Create a new daily note for today"*

> *"What have I written about improving my morning routine?"*

> *"List all the folders in my vault"*

> *"Read my weekly review template and create a new one for this week"*

Claude uses qmd's semantic search to find relevant notes (even when the exact words don't match), retrieves their content, and uses the filesystem server to create or modify notes.

## What's in the Box

```
obsidian-vault/
├── SKILL.md                        # Main skill instructions
├── CLAUDE-snippet.md               # Template to customize and add to ~/.claude/CLAUDE.md
├── references/
│   └── obsidian-syntax.md          # Obsidian Markdown syntax reference
└── README.md                       # You are here
```

## Troubleshooting

**qmd not finding anything:**
```bash
# Check what's indexed
qmd status

# Re-index and re-embed
qmd update
qmd embed

# Test a search directly
qmd search "test query"
```

**MCP servers not connecting:**
```bash
# Test qmd MCP directly
qmd mcp --help

# Test filesystem server directly
npx -y @modelcontextprotocol/server-filesystem "/path/to/your/Obsidian/Vaults"

# Check Claude Code registration
claude mcp list
```

**Claude Desktop not showing tools:**
- Verify your JSON config has no syntax errors (trailing commas are a common culprit)
- Check logs at `~/Library/Logs/Claude/` (macOS)
- Make sure the vault path is absolute (no `~` or `$HOME`)

**New notes not appearing in search:**
- qmd indexes are not live-updated. Run `qmd update && qmd embed` after creating notes to make them searchable.

## License

MIT — do whatever you want with it.
