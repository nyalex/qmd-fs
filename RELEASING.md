# Releasing

How to build the `.skill` file and publish a GitHub release.

## Repo layout

```
obsidian-vault/                     # skill source — tracked in git, edit this
├── SKILL.md                        # entry point
└── references/
    └── obsidian-syntax.md
CLAUDE-snippet.md                   # user config template (not part of the skill)
README.md                           # install and usage docs
obsidian-vault.skill                # build artifact — gitignored, never committed
```

`obsidian-vault/` is the source of truth. The `.skill` file is generated from it at
release time, so changes go in `obsidian-vault/SKILL.md` and get reviewed as a normal
diff. Do not edit the zip.

## What is a `.skill` file?

A zip archive that Claude Desktop and Claude Code load skill instructions from. The
archive must contain a single top-level directory named after the skill, with
`SKILL.md` at its root.

## Building

From the repo root:

```bash
rm -f obsidian-vault.skill
zip -r obsidian-vault.skill obsidian-vault/ -x '*.DS_Store'
```

Verify the structure before shipping:

```bash
unzip -l obsidian-vault.skill
```

You should see `obsidian-vault/SKILL.md` and `obsidian-vault/references/obsidian-syntax.md`
and nothing else. `README.md`, `CLAUDE-snippet.md`, and `RELEASING.md` are repo
documentation and must **not** be in the archive.

## Before tagging

- [ ] Every qmd tool named in `SKILL.md` and `README.md` exists in the qmd version you
      claim to support (`query`, `get`, `multi_get`, `status` as of qmd 2.x)
- [ ] No personal paths, vault contents, collection descriptions, or org names —
      `CLAUDE-snippet.md` is a template, not a copy of your config
- [ ] README's minimum qmd and Node versions still accurate
- [ ] Skill installs clean: `unzip obsidian-vault.skill -d /tmp/skilltest && cat /tmp/skilltest/obsidian-vault/SKILL.md`

## Creating a release

```bash
git tag v1.1.0
git push origin v1.1.0

rm -f obsidian-vault.skill
zip -r obsidian-vault.skill obsidian-vault/ -x '*.DS_Store'

gh release create v1.1.0 obsidian-vault.skill \
  --title "v1.1.0" \
  --notes "What changed in this release."
```

[Semantic versioning](https://semver.org): minor for new behavior, patch for fixes to
existing instructions. Bump the minor version when the skill starts depending on a
newer qmd.

Or through the UI: Releases → Draft a new release → pick the tag → attach
`obsidian-vault.skill` → publish.

## Installing from a release

Users download `obsidian-vault.skill` from the [releases page](../../releases) and
follow the README.
