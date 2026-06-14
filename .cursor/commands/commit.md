# Commit

You are preparing a **git commit** for the **israkir-prompts** repository — reusable coding-agent prompt templates (`prompts/`), catalog (`manifest.yaml`), CLI (`bin/israkir-prompt`), and project docs (`README.md`).

**Layout (what commits usually touch):**

- **`prompts/<category>/<slug>.md`** — one prompt per file; YAML frontmatter (`id`, `title`, `category`, `tags`, `version`) + Markdown body.
- **`manifest.yaml`** — registry of prompt ids and paths; **`id` must match** frontmatter in the corresponding file.
- **`bin/israkir-prompt`** — Bash CLI (`list`, `show`, `compose`, `path`); uses Python 3 stdlib only.
- **`README.md`** — usage, catalog table, conventions; update when adding prompts or changing workflow.

**Hooks:** There is **no** `.pre-commit-config.yaml` or git hook config in this repo today. If hooks are added later, read repo root for what actually runs on `git commit` / `git push` and follow that instead of assuming Copinance-style Makefile targets.

## Approval-first (default)

**Do not** run `git add` or `git commit` on the first response. The user invokes this command to **review a proposal** and approve before anything is recorded.

1. **Apply edits** only as needed to prepare the commit content. It is fine to write those file changes in the workspace so the user can see the diff.
2. **Do not** stage or commit until the user **explicitly approves** (e.g. “commit it”, “looks good, go ahead”, “run git commit”). After approval, stage and commit per the rules below.

If the user’s message already clearly says to commit now (not just to prepare), treat that as approval and proceed with staging + commit.

## Before committing (and for the proposal)

1. Show `git status -sb` and summarize what will be included.
2. List suggested `git add` paths or `git add -p` guidance when the change set is large or mixed.
3. **New or changed prompts:** confirm `manifest.yaml` is updated and frontmatter `id` matches the manifest entry. If the README prompt table is maintained manually, note whether it needs an update.

## Quality checks (suggest before commit)

From repo root:

- **`./bin/israkir-prompt list`** — manifest parses; all registered prompts resolve.
- **`./bin/israkir-prompt show <id>`** — smoke-test changed prompt ids (body prints, frontmatter stripped).
- **`bash -n bin/israkir-prompt`** — when the CLI script changed.
- **Prompt edits:** frontmatter present; `id` kebab-case; `version` bumped if scope or behavior changed materially.
- **`shellcheck bin/israkir-prompt`** — optional, if installed.

If `.pre-commit-config.yaml` exists at commit time, stage intended files and run **`pre-commit run`** (or **`pre-commit run --all-files`**) per that config.

## Commit message (for the proposal)

Propose a **detailed commit message** using [Conventional Commits](https://www.conventionalcommits.org/): a **subject** plus a **body**. Base the body on the actual diff—do not invent changes.

**Subject (first line)**

- Format: `type(scope): imperative summary` — max **~72 characters**, no trailing period.
- Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, etc.
- Scope examples: `prompts`, `ux`, `cli`, `manifest`, `readme`, `cursor`, `repo` — or omit if noisy.

**Body (following lines; required for this command)**

- Start after a blank line (Git will store subject + body correctly when using two `-m` flags or a here-doc).
- Write **3–8 short lines** (or more if the change is large), wrap near **72 chars** per line.
- Include, when relevant:
  - **What** changed (e.g. `prompts/ux/...`, `manifest.yaml`, `bin/israkir-prompt`, `README.md`, `.cursor/commands/`).
  - **Why** (problem, goal, or tradeoff).
  - **How to validate** (e.g. `./bin/israkir-prompt list`, `./bin/israkir-prompt show <id>`, `bash -n bin/israkir-prompt`) — only what applies.
  - **Risks / follow-ups** if any.
- You may include **GitHub closing keywords** as normal prose in the body (e.g. `Fixes #123`) when an issue exists.

**Forbidden — trailers in any form**

- Do **not** use `git commit --trailer ...` (including `Made-with: Cursor`, `Co-authored-by:`, `Signed-off-by:`, etc.).
- Do **not** paste or suggest a commit shell line that contains `--trailer`. Cursor or other tools may offer this; **omit it entirely**.
- Do **not** end the message with Git trailer lines (`Key: value` blocks after the body). A normal prose body is fine; GitHub closing lines like `Fixes #123` in the body are fine.

## After explicit user approval

Stage files (`git add …`) and create the commit with **only** `-m` or `-F` — **no other `git commit` flags** except `--no-verify` if the user explicitly bypasses hooks.

**Single-line subject + paragraph body (two arguments):**

```bash
git commit -m "<subject>" -m "<body>"
```

**Multiline message from stdin (no trailers):**

```bash
git commit -F - <<'EOF'
type(scope): short subject under 72 chars

First paragraph explaining what and why.

- Bullet if listing files or steps helps.
EOF
```

Do **not** prepend `--trailer` or any flag before `-F` or `-m`. The examples above are the full command.

If the commit fails on hooks, read the hook output, fix issues, and retry. Remind that **`git commit --no-verify` bypasses hooks** only if the user explicitly wants to skip checks.

## Output

**Proposal (default response):** status summary, full proposed commit subject + body, and **copy-paste-ready** example commands (`git add …`, then `git commit …`) so the user can run them locally. End with a short line inviting approval to run staging + commit from the agent if they prefer.

**After an approved commit:** show the full message with `git log -1` (or `git log -1 --format=medium`).
