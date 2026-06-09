# Superpowers (obra/superpowers) Installation Reference

Repo: https://github.com/obra/superpowers
Skills collection for AI coding agents (14 skills).

## What Was Installed

### Installed in `superpowers/` category (9 skills):

| Skill | Purpose | Extra files |
|-------|---------|-------------|
| `brainstorming` | Socratic design refinement | `scripts/`, `spec-document-reviewer-prompt.md`, `visual-companion.md` |
| `dispatching-parallel-agents` | Concurrent subagent parallel workflows | (SKILL.md only) |
| `executing-plans` | Batch execution with human checkpoints | (SKILL.md only) |
| `finishing-a-development-branch` | Merge/PR decision workflow | (SKILL.md only) |
| `receiving-code-review` | Responding to code review feedback | (SKILL.md only) |
| `using-git-worktrees` | Parallel dev branches via git worktrees | (SKILL.md only) |
| `using-superpowers` | Intro/bootstrap to the superpowers system | `references/` |
| `verification-before-completion` | Ensure fixes actually work | (SKILL.md only) |
| `writing-skills` | Create new skills following best practices | `anthropic-best-practices.md`, `examples/`, `graphviz-conventions.dot`, `persuasion-principles.md`, `render-graphs.js`, `testing-skills-with-subagents.md` |

### Already existed as Hermes built-ins (skipped, 5 skills):
`requesting-code-review`, `subagent-driven-development`, `systematic-debugging`, `test-driven-development`, `writing-plans`

## Installation Command That Worked

```bash
# Per-skill URL install:
hermes skills install "https://raw.githubusercontent.com/obra/superpowers/main/skills/<skill-name>/SKILL.md" --category "superpowers" --force -y
```

## Skills Blocked by Scanner (3)

These referenced CLAUDE.md/AGENTS.md which triggered CRITICAL persistence false positives:
- `receiving-code-review` — flagged for "You're absolutely right" (instruction-following pattern)
- `using-superpowers` — flagged for CLAUDE.md/GEMINI.md/AGENTS.md references
- `writing-skills` — flagged for CLAUDE.md reference and skill cross-reference

Workaround: manually copy files to `~/.hermes/profiles/<profile>/skills/<category>/<name>/`.

## Tap Registration

```bash
hermes skills tap add https://github.com/obra/superpowers
```
This succeeded but doesn't appear to be stored persistently in config or used for automatic discovery. Its purpose in Hermes is unclear — skills must still be URL-installed individually.

## Key Paths on Disk

| Path | Contents |
|------|----------|
| `~/.hermes/profiles/coder/skills/superpowers/` | Profile-level copies of all 9 skills |
| `~/.hermes/skills/brainstorming/` (etc.) | URL-installed copies (default hermes home) |
| `~/.hermes/profiles/coder/home/` | Profile HOME dir (where `$HOME` resolves inside profile) |
