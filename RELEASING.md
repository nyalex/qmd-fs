# Releasing

This guide covers how to build the `.skill` file and publish a GitHub release.

## What is a `.skill` file?

A `.skill` file is a zip archive that Claude Code and Claude.ai use to load skill instructions. It contains:

```
obsidian-vault/
├── SKILL.md                        # Main skill instructions (required)
└── references/
    └── obsidian-syntax.md          # Supporting reference material
```

The archive must use the skill name (`obsidian-vault/`) as the top-level directory. `SKILL.md` at the root of that directory is the entry point — Claude reads it to learn what the skill does and how to behave.

Files like `README.md` and `CLAUDE-snippet.md` are **not** included in the `.skill` file. They live in the repo for documentation and onboarding but aren't part of the installed skill.

## Building the `.skill` file

From the repo root:

```bash
zip -r obsidian-vault.skill obsidian-vault/
```

This zips the `obsidian-vault/` directory (which contains `SKILL.md` and `references/`) into `obsidian-vault.skill`.

Verify the contents look right:

```bash
unzip -l obsidian-vault.skill
```

You should see `obsidian-vault/SKILL.md` and any files under `obsidian-vault/references/`.

## Creating a GitHub release

### 1. Tag the release

```bash
git tag v1.0.0
git push origin v1.0.0
```

Use [semantic versioning](https://semver.org). Bump the minor version for new features or behavior changes, patch for fixes to existing instructions.

### 2. Build the `.skill` file

```bash
zip -r obsidian-vault.skill obsidian-vault/
```

### 3. Create the release on GitHub

```bash
gh release create v1.0.0 obsidian-vault.skill \
  --title "v1.0.0" \
  --notes "Description of what changed in this release."
```

Or create the release through the GitHub UI:

1. Go to the repo → **Releases** → **Draft a new release**
2. Choose the tag you just pushed
3. Add a title and release notes
4. Attach `obsidian-vault.skill` as a binary
5. Publish

## Installing from a release

Users download `obsidian-vault.skill` from the [releases page](../../releases) and follow the instructions in the README.
