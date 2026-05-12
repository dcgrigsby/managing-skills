# managing-skills-skill

A portable skill that teaches AI agents the user's conventions for installing, updating, and editing other agent skills. It captures three rules — always use `npx skills` (never tool-bundled installers), the canonical `-a` flag list for `add` / `update`, and edit-via-source (never hand-edit the installed copy).

This README is for humans setting the skill up. The agent-facing surface lives in [SKILL.md](SKILL.md).

## Why this exists

Skill management has a few easy-to-miss footguns:

- `npx skills add … -a '*'` symlinks the skill into every harness `npx skills` knows about — including ones you don't have installed — leaving dead directories that clutter the filesystem.
- Tool-bundled installers (`<some-tool> skill install …`) write files into `~/.agents/skills/` and `~/.claude/skills/` without updating `~/.agents/.skill-lock.json`, leaving orphans that `npx skills` can't update or remove.
- Hand-editing `~/.agents/skills/<name>/SKILL.md` directly looks like it works — until the next `npx skills update` clobbers it. If the push to the source repo silently failed, the lock hash drifts from reality and you can't tell.

These aren't theoretical — the rules in [SKILL.md](SKILL.md) come from incidents we hit.

## Install

```bash
npx skills add dcgrigsby/managing-skills-skill -g -a claude-code -a gemini-cli -a codex -y
```

Or clone this repo and point your skill-loader at the directory.

## Scope

This skill owns:

- How to install a skill (always via `npx skills add` with explicit `-a` flags)
- How to update a skill (always via `npx skills update`, same flags)
- How to edit a skill (source repo + push + `npx skills update`, never hand-edit the installed copy)
- How to create a new skill end-to-end (local repo → GitHub → install)
- How to remove a skill

It does NOT own:

- The content or behavior of any other skill (those are owned by their own repos)
- Skill authoring guidance (see `obra/superpowers/skills/writing-skills` for that)
- Discovery / search for new skills to install (see `vercel-labs/skills/skills/find-skills`)

## License

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
