# israkir-prompts

Reusable prompt templates for coding agents (Cursor, Claude Code, Copilot, etc.). Each template is a standalone Markdown file you can reference from any project.

**Cursor:** run **`/commit`** (`.cursor/commands/commit.md`) to propose a Conventional Commit before staging; approve explicitly before anything is recorded.

## Quick start

```bash
# From this repo
./bin/israkir-prompt list
./bin/israkir-prompt show timestamp-data-source-audit

# Optional: add CLI to PATH
export PATH="$PATH:/path/to/israkir-prompts/bin"
israkir-prompt show timestamp-data-source-audit | pbcopy
```

**In Cursor** — add this repo to your workspace (or clone it anywhere) and attach a prompt in chat:

```text
@prompts/ux/timestamp-data-source-audit.md

Apply this prompt to the current project. Work through Phase A first, then propose changes before implementing.
```

**In Claude**

*Claude Code* — run from your target project directory. Point Claude at a prompt file by path (absolute or relative if both repos are open):

```text
Read /path/to/israkir-prompts/prompts/ux/timestamp-data-source-audit.md and treat it as the full task brief.

Apply it to this repository. Work through Phase A first, then propose changes before implementing.
```

Paste works too — pipe or copy from the CLI:

```bash
israkir-prompt show timestamp-data-source-audit | pbcopy   # macOS
israkir-prompt show timestamp-data-source-audit | xclip -selection clipboard   # Linux
```

For repeat use, add a one-liner to the target project’s `CLAUDE.md` (optional):

```markdown
For timestamp/provenance UX audits, load `~/git/israkir-prompts/prompts/ux/timestamp-data-source-audit.md` and follow it as the task brief.
```

*Claude.ai (Projects)* — add prompt `.md` files as **Project knowledge**, or paste the output of `israkir-prompt show <id>` into a new conversation. Keep project instructions short; put the full brief in knowledge or the first message.

**Multiple prompts:**

*Cursor:*
```text
@prompts/ux/timestamp-data-source-audit.md
@prompts/other-category/other-prompt.md

Apply both prompts to this codebase in order.
```

*Claude Code:*
```text
Read these files from israkir-prompts and apply them to this repo in order:
1. /path/to/israkir-prompts/prompts/ux/timestamp-data-source-audit.md
2. /path/to/israkir-prompts/prompts/other-category/other-prompt.md
```

Or use the CLI:

```bash
israkir-prompt compose timestamp-data-source-audit other-id
```

## Repository layout

```text
israkir-prompts/
├── manifest.yaml          # Catalog (ids, paths, metadata for tooling)
├── bin/israkir-prompt     # list | show | compose | path
├── .cursor/commands/      # Cursor slash commands (e.g. /commit)
└── prompts/
    └── <category>/
        └── <slug>.md      # One prompt per file, YAML frontmatter + body
```

| Path | Purpose |
|------|---------|
| `prompts/<category>/<slug>.md` | Prompt content agents consume |
| `manifest.yaml` | Stable ids for CLI and automation |
| `bin/israkir-prompt` | Print prompts without opening files manually |

## Using prompts in any project

You do **not** need to copy prompts into target repos.

1. **Reference by path** — Keep this repo on disk; `@` the file in Cursor, or ask Claude Code to read the path (multi-root workspace or absolute path both work).
2. **Pipe from CLI** — `israkir-prompt show <id>` and paste, or pipe to clipboard (`pbcopy` / `xclip`).
3. **Project knowledge (Claude.ai)** — Upload selected `prompts/**/*.md` files to a Project.
4. **Submodule / sparse checkout** — Pin a version in larger monorepos if you want prompts vendored.

Suggested agent instruction wrapper (paste after the prompt):

```text
Apply the above prompt to the repository I have open.
Follow the phases in order. Inventory before code changes.
Ask before large refactors outside the stated scope.
```

## Adding a prompt

1. Create `prompts/<category>/<slug>.md` with frontmatter:

```yaml
---
id: my-prompt-id
title: Human-readable title
category: ux
tags: [tag1, tag2]
version: 1.0.0
---
```

2. Register it in `manifest.yaml` (same `id` as frontmatter).
3. Run `./bin/israkir-prompt list` to verify.

**Conventions**

- `id`: kebab-case, stable forever once published
- `category`: broad grouping (`ux`, `security`, `refactoring`, …)
- Body: written as a complete task brief an agent can execute without extra context
- Bump `version` in frontmatter when behavior or scope changes materially

## Prompts

| ID | Category | Title |
|----|----------|-------|
| `timestamp-data-source-audit` | ux | Timestamp & Data-Source UX Consistency Audit |

Run `./bin/israkir-prompt list` for the current catalog.

## License

See [LICENSE](LICENSE).
