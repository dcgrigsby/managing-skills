---
name: managing-skills
description: Use whenever the user asks to install, update, remove, or edit an agent skill on their machine, or when a tool offers to install its own bundled skill. Triggers on phrases like "install this skill", "add a skill", "update X skill", "edit the X skill to do Y", "change behavior of the X skill", "the foo CLI says I can `foo skill install` — should I?", or any time the agent is about to modify a file under `~/.agents/skills/` or `~/.claude/skills/`. Owns the user's conventions for how skills get installed (always via `npx skills`, never via tool-bundled installers) and how they get edited (source repo + push + `npx skills update`, never hand-edit the installed copy). Skip when the work is purely *using* a loaded skill — only use when the request is about the skill itself as an artifact (install / update / remove / edit / inspect).
---

# Managing skills

This skill owns the user's conventions for managing agent skills on their machine — installs, updates, removals, and edits. It exists because skill management has a few easy-to-miss footguns (`-a '*'`, tool-bundled installers, hand-editing the installed copy) that quietly break either the install graph or the source-of-truth contract. Follow the rules below before touching anything under `~/.agents/skills/` or `~/.claude/skills/`.

## Capturing new skill-management feedback

If the user gives you a new rule, correction, or preference about how skills get installed / updated / edited (or about meta-tooling like `npx skills`, lockfile management, harness targets), **add it to this SKILL.md** via the edit-via-source flow (Rule 3 below) — not to auto-memory. Auto-memory is harness-specific (Claude Code only) and won't travel to Codex CLI, Gemini CLI, or other harnesses; this skill will. The rules in this file were themselves migrated out of auto-memory for exactly that reason; don't recreate the problem.

The same goes for new conventions about *creating* skills (boilerplate, repo layout, README structure, `gh repo create` flags): they belong here, in the relevant "End-to-end flow" section.

## Layout on disk

- `~/.agents/skills/<name>/` — canonical install location. The actual files live here.
- `~/.claude/skills/<name>` — symlink into `~/.agents/skills/<name>/` for Claude Code. Created by `npx skills` for harnesses that don't read `~/.agents/skills/` directly.
- `~/.agents/.skill-lock.json` — tracks which skills are installed, the GitHub source for each, and the commit hash. `npx skills update` reads this to know what to pull.
- `~/<name>-skill/` — the user's local source repo for skills they author or maintain (`personal-workflow-skill`, `omnifocus-skill`, `obsidian-skill`, `slack-skill`, `screenshot-skill`, `autoresearch-skill`, `managing-skills-skill`, etc.). These are the **only** files to edit when changing skill behavior.

Use `~/.agents/.skill-lock.json` to find a skill's source repo:

```bash
jq '.skills["<name>"].sourceUrl' ~/.agents/.skill-lock.json
```

## Rule 1 — Install via `npx skills`, never via tool-bundled installers

When a CLI ships its own skill installer (`qmd skill install`, `<tool> skill install`, "auto-install" flows from tools that bundle their skills), **do not run it**. Surface it as a suggestion to the user, who installs via the canonical `npx skills add` flow.

**Why:** All filesystem skills go through `npx skills` so they're tracked in `~/.agents/.skill-lock.json`. Tool-specific installers write files into `~/.agents/skills/` and `~/.claude/skills/` *without* updating the lockfile, leaving orphans that `npx skills` can't update, list, or remove cleanly.

**How to apply:**

- A CLI's `<tool> skill install` invitation is a *recommendation* to relay, not an instruction to execute. Tell the user the skill is available; let them install it through `npx skills`.
- The skill content is usually viewable via `<tool> skill show` (or analogous). Read it for context if helpful — but don't install it.
- If the skill is bundled inside an npm package (not a standalone GitHub repo), say so — `npx skills add` typically takes a GitHub repo, so the install pattern may differ. Ask before proceeding.

## Rule 2 — Canonical `npx skills` install / update command

Always pass explicit `-a` flags for the user's target harnesses. **Never use `-a '*'`.**

```bash
npx skills add <github-repo> -g -a claude-code -a gemini-cli -a codex -y
```

Same flags for `update`:

```bash
npx skills update <name> -g -a claude-code -a gemini-cli -a codex -y
```

**Why:** `-a '*'` creates symlink directories for every harness `npx skills` knows about — including ones the user doesn't have installed — which clutter the filesystem with dead directories. The three named harnesses (`claude-code`, `gemini-cli`, `codex`) are the only ones the user actively runs that consume filesystem skills.

**How to apply:**

- Codex CLI and Gemini CLI are "universal-reader" agents in the `npx skills` model — they read directly from `~/.agents/skills/`. The install writes once and they pick it up without per-agent symlinks. They still need to appear in `-a` so `npx skills` records them as targets.
- Claude Code needs explicit symlinks, which `npx skills` creates under `~/.claude/skills/`.
- `npx skills list -g` may show only "Claude Code" under the `Agents:` line for these skills — that display only counts symlinked agents and omits universal-readers. Codex and Gemini still see them. Display quirk, not a bug.
- Other harnesses the user has (Claude Desktop, ChatGPT, Gemini Desktop, Codex Desktop) don't load filesystem skills — they use MCP servers, plugins, or custom GPTs and are not targets for `npx skills`.
- If the user adds or removes a target harness (e.g., installs a new CLI that uses filesystem skills, or uninstalls one), update the `-a` flag list accordingly.

## Rule 3 — Edit installed skills via source repo, never directly

When the user asks for a behavior change to a skill installed under `~/.agents/skills/<name>/`, edit the **source repo** (`~/<name>-skill/SKILL.md` or wherever the source lives), commit, push, then run `npx skills update`. **Do not also edit the file under `~/.agents/skills/<name>/`.**

**Why:** `npx skills update` pulls from the GitHub source recorded in `~/.agents/.skill-lock.json` and rewrites the installed copy from that commit. Direct edits to the installed file:

1. Get clobbered on the next legitimate update.
2. Mask whether the source-and-push round-trip actually succeeded — a failed push followed by `npx skills update` reports "up to date" (no-op against the unchanged GitHub head) while the installed copy silently still reflects the local hand-edit. The lock hash drifts from reality.

**How to apply:**

- Source-only edits. Let `npx skills update` do the propagation.
- If the push fails, stop and surface that. Don't paper over it by hand-writing the installed copy.
- Use the canonical update invocation from Rule 2 (`-a claude-code -a gemini-cli -a codex`, never `-a '*'`).
- To find the source repo for an installed skill, check `~/.agents/.skill-lock.json` → `sourceUrl`:

  ```bash
  jq -r '.skills["<name>"].sourceUrl' ~/.agents/.skill-lock.json
  ```

- The corresponding local clone is usually at `~/<name>-skill/`. If it's missing, clone it from `sourceUrl` before editing.

### Recovery: skill exists on disk but isn't in the lockfile

If `npx skills update <name>` returns `No installed skills found matching: <name>`, the skill is installed under `~/.agents/skills/<name>/` but isn't tracked in `~/.agents/.skill-lock.json`. Common causes: installed before the lockfile existed, manually `git clone`'d, or installed by a tool-bundled installer in violation of Rule 1. **Recovery: `npx skills add <repo> -g -a claude-code -a gemini-cli -a codex -y`** — this overwrites the on-disk copy and registers the entry, after which `update` works normally.

Detect up front before relying on `update`:

```bash
jq -e '.skills["<name>"]' ~/.agents/.skill-lock.json >/dev/null && echo tracked || echo untracked
```

## Rule 4 — Keep `description:` under 1024 characters

Codex CLI (and Gemini CLI, by the same loader contract) enforces a **1024-character limit** on the `description:` field in the SKILL.md frontmatter. **Skills that exceed it are silently skipped at startup** — Codex prints a one-line warning before its banner (`Skipped loading 1 skill(s) … invalid description: exceeds maximum length of 1024 characters`) but the skill simply isn't loaded. Claude Code does not enforce this limit, so an oversized description can ship and quietly disappear from the other harnesses without anything in the Claude Code session noticing.

**Why:** Hard limit in the loader — no truncation, no fallback. Easy to miss because (a) `npx skills add` / `update` happily installs an oversized description without warning, and (b) Claude Code keeps working, so the user sees the skill there and assumes it's fine everywhere.

**How to apply:**

- Before committing any SKILL.md change that touches the description, check the length:

  ```bash
  python3 -c "import yaml,sys; t=open(sys.argv[1]).read(); e=t.find('\n---',3); d=yaml.safe_load(t[3:e])['description']; print(len(d), '/', 1024)" SKILL.md
  ```

- Aim for ≤ ~1000 chars to leave headroom. When trimming, preserve every trigger phrase and disambiguator — compress prose instead (drop redundant examples, tighten "Designed to run from inside the X — does not support Y" to "Run from X; no Y", collapse parenthetical lists).
- After install / update, verify Codex loads it cleanly:

  ```bash
  codex exec "ok" 2>&1 | grep -iE 'skipped|invalid description|exceeds'
  ```

  Empty output = clean. Any hit = still rejected; re-fix the source and re-run the edit flow.
- Audit all installed skills at once:

  ```bash
  cd ~/.agents/skills && python3 -c "
  import os, yaml
  for d in sorted(os.listdir('.')):
      f = os.path.join(d, 'SKILL.md')
      if not os.path.isfile(f): continue
      txt = open(f).read()
      if not txt.startswith('---'): continue
      end = txt.find('\n---', 3)
      desc = (yaml.safe_load(txt[3:end]) or {}).get('description','') or ''
      if len(desc) > 1024:
          print(f'OVER {len(desc):5d} {d}')
  "
  ```

## End-to-end flow for a skill edit

For "change the X skill so it does Y":

1. Identify source: `jq -r '.skills["X"].sourceUrl' ~/.agents/.skill-lock.json`.
2. Verify the local clone exists at `~/<X>-skill/`. If not, `git clone <sourceUrl> ~/<X>-skill/`.
3. Edit `~/<X>-skill/SKILL.md` (or other files in the repo).
4. Commit with a descriptive message; push to the origin. **If push fails, stop and surface the failure.**
5. Run `npx skills update <X> -g -a claude-code -a gemini-cli -a codex -y`. If it reports `No installed skills found matching: <X>`, fall through to `npx skills add <repo> -g -a claude-code -a gemini-cli -a codex -y` (see "Recovery: skill exists on disk but isn't in the lockfile" above).
6. Verify the installed file under `~/.agents/skills/<X>/` reflects the new content.
7. If the edit touched the `description:` field, run `codex exec "ok" 2>&1 | grep -iE 'skipped|invalid description|exceeds'` — empty output confirms Codex accepted the new description (see Rule 4).

## End-to-end flow for a new skill

For "create a new skill called X":

1. Create `~/<X>-skill/` with at least:
   - `SKILL.md` — frontmatter (`name`, `description`) + body.
   - `LICENSE` — match other repos (Apache 2.0).
   - `NOTICE` — copyright + any safety warnings.
   - `README.md` — human-facing intro + install command.
   - `.gitignore` — at minimum `.DS_Store`, `__pycache__/`, `.venv/`.
2. `git init`, commit.
3. `gh repo create <user>/<X>-skill --public --source ~/<X>-skill --push` (default visibility is public to match the other repos; private only if the user explicitly asks).
4. Install: `npx skills add <user>/<X>-skill -g -a claude-code -a gemini-cli -a codex -y`.
5. Verify the install lands at `~/.agents/skills/<X>/` and the lockfile entry exists.

## Removing a skill

`npx skills remove <name>` updates the lockfile and removes the install. If the user wants to also delete the local source clone, do that separately (`rm -rf ~/<name>-skill/`) and confirm before destructive removal.

## When the canonical paths don't apply

A few escape hatches are legitimate but should be surfaced clearly:

- **One-off test / experimentation** with a draft skill that isn't ready for a public repo: cloning into `~/.agents/skills/<name>/` manually works, but tell the user it bypasses `npx skills` tracking and will need a proper install before it's portable.
- **Forks of someone else's skill**: edit the fork, push to the fork's remote, and update the lockfile entry's `sourceUrl` (or reinstall from the fork URL). Hand-editing the installed copy is still wrong.
- **Plugin-style installs** (e.g., `anthropics/skills` with `pluginName: example-skills` in the lockfile): these come from monorepos with multiple skills. Still updated via `npx skills update`, not direct edits.
