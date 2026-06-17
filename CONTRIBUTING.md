# Contributing to israkir-prompts

Thank you for helping improve reusable coding-agent prompt templates. This document covers how to add or change prompts, keep the catalog in sync, and open pull requests.

---

## What belongs in this repo

| In scope | Out of scope |
|----------|--------------|
| Standalone `.md` prompt briefs agents can run without extra context | One-off chat snippets tied to a single private project |
| CLI and manifest changes that support discovery/composition | Heavy automation frameworks or agent runtimes |
| Cursor slash commands for **this** repo (e.g. `/commit`) | Copying prompts into consumer repos when path reference works |
| README / CONTRIBUTING updates when behavior or catalog changes | Secrets, API keys, or environment-specific paths in committed files |

---

## Adding a new prompt

### 1. Create the prompt file

Path: `prompts/<category>/<slug>.md`

**Frontmatter** (required):

```yaml
---
id: my-prompt-id
title: Human-readable title
category: ux
tags: [tag1, tag2]
version: 1.0.0
---
```

**Body** — write a complete task brief:

- **Role** — who the agent is pretending to be
- **Scope** — in/out of scope, explicit boundaries
- **Phases** — for audits, prefer **Phase A (discovery) → Phase B (plan & implement) → Phase C (verify)**
- **Acceptance criteria** — how the agent knows it is done
- **Constraints** — ask before large refactors; inventory before code changes

Adapt the structure of existing prompts in the same category.

### 2. Register in `manifest.yaml`

Add an entry under `prompts:` with the **same `id`** as frontmatter:

```yaml
  - id: my-prompt-id
    path: prompts/ux/my-prompt-id.md
    title: Human-readable title
    category: ux
    tags: [tag1, tag2]
```

### 3. Update documentation

- Add a row to the **Prompt Catalog** table in `README.md`.
- Run `./bin/israkir-prompt list` and `./bin/israkir-prompt show my-prompt-id` to verify resolution.

### 4. Versioning

- New prompt: start at `version: 1.0.0`.
- Material scope or behavior change: bump minor (`1.1.0`) or major (`2.0.0`) in frontmatter.
- Typos / clarifications only: patch (`1.0.1`).

**Never rename or reuse a published `id`** — downstream automation and `CLAUDE.md` one-liners depend on stable ids.

---

## Conventions

| Field | Rule |
|-------|------|
| `id` | kebab-case, stable forever once published |
| `category` | Broad grouping: `ux`, `api`, `dev`, `security`, `refactoring`, … |
| `tags` | Lowercase, searchable keywords |
| File path | `prompts/<category>/<slug>.md` — slug usually matches `id` |
| Body | Self-contained; no “see our internal wiki” dependencies |

**Prompt style**

- Prefer questions and suggestions over commands when reviewing human-written code (audit prompts).
- Separate what agents should catch vs. what linters/CI already enforce.
- Use tables and checklists for inventories; use phased headings for execution order.

---

## Changing existing prompts

1. Edit `prompts/<category>/<slug>.md`.
2. Bump `version` in frontmatter if behavior or scope changed materially.
3. Update `manifest.yaml` only if `id`, `path`, `title`, `category`, or `tags` changed.
4. Note breaking changes in the PR description (e.g. removed phases, narrower scope).

---

## CLI (`bin/israkir-prompt`)

The CLI uses Python 3 stdlib only. If you change commands or manifest parsing:

- Update `usage()` in `bin/israkir-prompt`.
- Smoke-test: `list`, `show <id>`, `compose <id> <id>`, `path <id>`.
- Document new subcommands in `README.md`.

---

## Cursor commands

Slash commands live in `.cursor/commands/`. If you add one:

- Use a short, action-oriented name (`commit.md` → `/commit`).
- Document it in `README.md` under Usage.
- Keep commands **approval-first** when they run git write operations.

---

## Pull request checklist

Before opening a PR:

- [ ] `id` in frontmatter matches `manifest.yaml`
- [ ] `./bin/israkir-prompt list` shows the new/changed entry
- [ ] `./bin/israkir-prompt show <id>` prints the expected body (no frontmatter in output)
- [ ] README catalog table updated if catalog changed
- [ ] `version` bumped when prompt behavior/scope changed materially
- [ ] No secrets or machine-specific absolute paths committed

**Commit messages** — [Conventional Commits](https://www.conventionalcommits.org/) preferred:

```
feat(prompts): add security-headers-audit template
fix(api): clarify hypermedia examples in restful-api-design-audit
docs: update README usage for Claude.ai Projects
```

In this repo you can run **`/commit`** in Cursor to draft a Conventional Commit proposal (approve explicitly before staging).

---

## Ideas welcome

- New categories: security, refactoring, testing, documentation, observability
- Framework-specific variants (e.g. Next.js timestamp patterns, FastAPI REST audit)
- Additional CLI helpers (validate manifest, count lines, lint frontmatter)
- Translations of prompt bodies (separate files or clearly labeled sections)
- More Cursor slash commands for maintainers

---

## Questions

Open a [GitHub issue](https://github.com/israkir/israkir-prompts/issues) for design questions before large prompts. For small fixes, a PR with a short description is fine.

---

## License

By contributing, you agree that your contributions are licensed under the same [GPL-2.0](LICENSE) license as the project.
