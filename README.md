<div align="center">

<h1>&#128221; israkir-prompts</h1>

<p>
  <strong>Reusable, production-ready prompt templates for coding agents</strong>
</p>

<p>
  <a href="https://github.com/israkir/israkir-prompts/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/License-GPL--2.0-22c55e?style=flat-square" alt="License: GPL-2.0"/>
  </a>
  <img src="https://img.shields.io/badge/Cursor-Ready-7c3aed?style=flat-square&logo=cursor&logoColor=white" alt="Cursor Ready"/>
  <img src="https://img.shields.io/badge/Claude_Code-Compatible-7c3aed?style=flat-square&logo=anthropic&logoColor=white" alt="Claude Code Compatible"/>
  <img src="https://img.shields.io/badge/Total_Lines-1%2C000%2B-3b82f6?style=flat-square" alt="1000+ lines"/>
  <img src="https://img.shields.io/badge/Prompts-3-f59e0b?style=flat-square" alt="3 prompts"/>
  <img src="https://img.shields.io/badge/PRs-Welcome-ec4899?style=flat-square" alt="PRs Welcome"/>
</p>

<p>
  <a href="#what-is-this">Overview</a>
  &middot;
  <a href="#usage">Usage</a>
  &middot;
  <a href="./CONTRIBUTING.md">Contributing</a>
</p>

</div>

---

<a name="what-is-this"></a>

### What is this?

**israkir-prompts** is a catalog of standalone Markdown prompt templates for [Cursor](https://cursor.com), [Claude Code](https://claude.ai/code), Copilot, and other coding agents. Each file is a complete task brief an agent can execute without extra context.

Prompts are **versioned**, **categorized**, and registered in `manifest.yaml` — discoverable via a small CLI (`bin/israkir-prompt`) or referenced directly by path in chat.

---

### &#10024; Key Features

- **Standalone prompts** — One `.md` file per task; YAML frontmatter + full brief body; no framework lock-in.
- **Stable IDs** — Kebab-case `id` in frontmatter and `manifest.yaml` for CLI, automation, and `@` references.
- **Three-phase workflow** — Audit prompts follow **Discovery → Plan & implement → Verify** so agents inventory before changing code.
- **CLI helpers** — `list`, `show`, `compose`, and `path` to print or pipe prompts without opening files manually.
- **Agent-agnostic** — Works with Cursor `@` attachments, Claude Code file reads, clipboard paste, or project knowledge.
- **Cursor `/commit`** — Built-in command (`.cursor/commands/commit.md`) for Conventional Commit proposals with approval-first staging.

---

### &#127760; Prompt Catalog

<table>
  <thead>
    <tr>
      <th>Category</th>
      <th>ID</th>
      <th>Title</th>
      <th>File</th>
      <th>Lines</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="1"><strong>UX</strong></td>
      <td><code>timestamp-data-source-audit</code></td>
      <td>&#128337; Timestamp &amp; Data-Source UX Consistency Audit</td>
      <td><code>prompts/ux/timestamp-data-source-audit.md</code></td>
      <td>~210</td>
    </tr>
    <tr>
      <td rowspan="1"><strong>API</strong></td>
      <td><code>restful-api-design-audit</code></td>
      <td>&#128268; RESTful API Design Audit &amp; Implementation</td>
      <td><code>prompts/api/restful-api-design-audit.md</code></td>
      <td>~385</td>
    </tr>
    <tr>
      <td rowspan="1"><strong>Dev</strong></td>
      <td><code>new-project-setup-makefile</code></td>
      <td>&#9881;&#65039; New Project Setup — Makefile Conventions + Production Integration</td>
      <td><code>prompts/dev/new-project-setup-makefile.md</code></td>
      <td>~435</td>
    </tr>
  </tbody>
</table>

Run `./bin/israkir-prompt list` for the live catalog.

---

### &#128260; Three-Phase Agent Workflow

Audit-style prompts share a consistent execution model:

```
Phase A — Discovery (inventory first)
  Map scope, find every relevant surface, document before editing
                    |
                    v
Phase B — Plan & implement
  Propose changes, get alignment, then implement in bounded steps
                    |
                    v
Phase C — Verify
  Check acceptance criteria, regressions, and documentation updates
```

---

### &#128193; Repository Structure

```
israkir-prompts/
|
+-- manifest.yaml                         # Catalog (ids, paths, metadata for tooling)
+-- README.md
+-- CONTRIBUTING.md
+-- LICENSE
+-- bin/
|   +-- israkir-prompt                    # CLI: list | show | compose | path
|
+-- prompts/
|   +-- ux/
|   |   +-- timestamp-data-source-audit.md
|   +-- api/
|   |   +-- restful-api-design-audit.md
|   +-- dev/
|       +-- new-project-setup-makefile.md
|
+-- .cursor/
    +-- commands/
        +-- commit.md                     # /commit — Conventional Commit helper
```

| Path | Purpose |
|------|---------|
| `prompts/<category>/<slug>.md` | Prompt content agents consume |
| `manifest.yaml` | Stable ids for CLI and automation |
| `bin/israkir-prompt` | Print prompts without opening files manually |

---

### &#128640; Installation

**Clone anywhere on disk** (no install step required):

```bash
# macOS / Linux
git clone https://github.com/israkir/israkir-prompts.git ~/git/israkir-prompts

# Windows (PowerShell)
git clone https://github.com/israkir/israkir-prompts.git "$env:USERPROFILE\git\israkir-prompts"
```

**Optional — add CLI to PATH:**

```bash
export PATH="$PATH:$HOME/git/israkir-prompts/bin"
israkir-prompt list
```

**In Cursor** — add this repo to your workspace (multi-root) or reference files by absolute path.

---

<a name="usage"></a>

### &#128161; Usage

#### Quick start

```bash
# From this repo
./bin/israkir-prompt list
./bin/israkir-prompt show timestamp-data-source-audit

# Optional: add CLI to PATH (see Installation)
israkir-prompt show timestamp-data-source-audit | pbcopy   # macOS
israkir-prompt show timestamp-data-source-audit | xclip -selection clipboard   # Linux
```

**Cursor:** run **`/commit`** (`.cursor/commands/commit.md`) in this repo to propose a Conventional Commit before staging; approve explicitly before anything is recorded.

#### CLI

```bash
./bin/israkir-prompt list
./bin/israkir-prompt show timestamp-data-source-audit
./bin/israkir-prompt path timestamp-data-source-audit
./bin/israkir-prompt compose timestamp-data-source-audit restful-api-design-audit
```

| Command | Purpose |
|---------|---------|
| `list` | Print all registered prompt ids, categories, titles |
| `show <id>` | Print prompt body (no YAML frontmatter) |
| `path <id>` | Print absolute path to the `.md` file |
| `compose <id> [id...]` | Print multiple prompts separated for sequential agent use |

**Clipboard piping:**

```bash
israkir-prompt show timestamp-data-source-audit | pbcopy                        # macOS
israkir-prompt show timestamp-data-source-audit | xclip -selection clipboard    # Linux
israkir-prompt path timestamp-data-source-audit | pbcopy                          # copy file path
```

#### In Cursor

Add this repo to your workspace (multi-root) or reference files by absolute path. Attach a prompt in chat:

```text
@prompts/ux/timestamp-data-source-audit.md

Apply this prompt to the current project. Work through Phase A first, then propose changes before implementing.
```

**Multiple prompts (Cursor):**

```text
@prompts/ux/timestamp-data-source-audit.md
@prompts/api/restful-api-design-audit.md

Apply both prompts to this codebase in order.
```

#### In Claude Code

Run from your **target project** directory. Point Claude at a prompt file by path (absolute, or relative if both repos are open):

```text
Read /path/to/israkir-prompts/prompts/ux/timestamp-data-source-audit.md and treat it as the full task brief.

Apply it to this repository. Work through Phase A first, then propose changes before implementing.
```

**Paste from CLI** — pipe or copy from `show` / `compose` (see clipboard commands above).

**Repeat use (optional)** — add a one-liner to the target project’s `CLAUDE.md`:

```markdown
For timestamp/provenance UX audits, load `~/git/israkir-prompts/prompts/ux/timestamp-data-source-audit.md` and follow it as the task brief.
```

**Multiple prompts (Claude Code):**

```text
Read these files from israkir-prompts and apply them to this repo in order:
1. /path/to/israkir-prompts/prompts/ux/timestamp-data-source-audit.md
2. /path/to/israkir-prompts/prompts/api/restful-api-design-audit.md
```

Or use the CLI:

```bash
israkir-prompt compose timestamp-data-source-audit restful-api-design-audit
```

#### Claude.ai (Projects)

Add prompt `.md` files as **Project knowledge**, or paste the output of `israkir-prompt show <id>` into a new conversation. Keep project instructions short; put the full brief in knowledge or the first message.

#### Copilot and other agents

Paste the output of `israkir-prompt show <id>` or attach the `.md` file if your tool supports file context. Use the suggested wrapper below.

#### Using prompts in any project

You do **not** need to copy prompts into target repos.

1. **Reference by path** — Keep this repo on disk; `@` the file in Cursor, or ask Claude Code to read the path (multi-root workspace or absolute path both work).
2. **Pipe from CLI** — `israkir-prompt show <id>` and paste, or pipe to clipboard (`pbcopy` / `xclip`).
3. **Project knowledge (Claude.ai)** — Upload selected `prompts/**/*.md` files to a Project.
4. **Submodule / sparse checkout** — Pin a version in larger monorepos if you want prompts vendored:

```bash
git submodule add https://github.com/israkir/israkir-prompts.git tools/israkir-prompts
# or sparse-checkout only prompts/ if your git workflow supports it
```

**Suggested wrapper** (paste after any prompt):

```text
Apply the above prompt to the repository I have open.
Follow the phases in order. Inventory before code changes.
Ask before large refactors outside the stated scope.
```

#### Example prompts

| Prompt | What happens |
|--------|-------------|
| `timestamp-data-source-audit` | Audits every user-visible timestamp; pairs time with data source; fixes timezone and freshness semantics |
| `restful-api-design-audit` | Reviews HTTP APIs against REST maturity (Level 2–3); resources, verbs, hypermedia, OpenAPI |
| `new-project-setup-makefile` | Scaffolds Makefile targets, local dev (host + container datastores), CI/registry production flow |
| `israkir-prompt compose <id>...` | Concatenates multiple prompts for sequential agent execution |

---

### &#128300; Highlights by Prompt

<details>
<summary><strong>&#128337; Timestamp &amp; Data-Source UX Audit</strong></summary>

- Distinguishes **provider time**, **server/cache time**, **client receive time**, and **user activity time**
- Covers UI labels: "Live", "Real-time", "Cached", "Updated", tooltips, and screen-reader text
- Phase A inventory of every user-visible timestamp before any code change
- Consistent pairing of **when** with **where the data came from**
- User timezone respect and page-level provenance priority

</details>

<details>
<summary><strong>&#128268; RESTful API Design Audit</strong></summary>

- **REST maturity model** — target Level 2 for internal APIs, Level 3 (hypermedia) for client-facing APIs
- Resource-oriented URIs, correct HTTP verbs and status codes, stateless requests
- OpenAPI / schema alignment, pagination, filtering, error representation (`ProblemDetail`)
- Hypermedia via links, `Link` headers, or standard profiles — not hardcoded URL construction
- Phase A discovery of existing routes and contracts before refactoring

</details>

<details>
<summary><strong>&#9881;&#65039; New Project Setup (Makefile)</strong></summary>

- Canonical targets: `be-*`, `fe-*`, `storage-*`, `worker-dev` across repos
- **Local dev** — app on host with hot reload; only datastores in containers (`podman-compose`)
- **Production** — CI builds images → registry (e.g. GHCR) → host pulls; no local builds on server
- Single root `.env` from `.env.example`; deprecated aliases as thin forwards
- Monorepo layout, CI wiring, and onboarding checklist for new apps

</details>

---

### &#129309; Contributing

Contributions are welcome! See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full guide: adding prompts, manifest sync, versioning, PR checklist, and conventions.

**Quick summary:**

1. Create `prompts/<category>/<slug>.md` with YAML frontmatter (`id`, `title`, `category`, `tags`, `version`).
2. Register the same `id` in `manifest.yaml`.
3. Update the README catalog table.
4. Run `./bin/israkir-prompt list` and `show <id>` to verify.

**Ideas:** new categories (security, refactoring, testing), framework-specific audit variants, CLI validators, more Cursor slash commands.

---

### &#128196; License

GPL-2.0 &copy; [israkir](https://github.com/israkir)

---

<div align="center">
  Made with &#10084;&#65039; for developers who ship consistent agent workflows
</div>
