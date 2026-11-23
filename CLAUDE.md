# skill-hide: project guide for Claude

## What this is

An **agent skill** (not a CLI) that lets a developer use AI skills privately in a shared team repo. The agent installs or adopts skills, hides them and everything they write from git and teammates, keeps them out of `skills-lock.json`, and checks before commits.

The owner prefers **very simple, plain explanations**, step by step, no jargon. Explain what changed and why in a few short sentences.

## Files

| File | Job |
|---|---|
| `skills/skill-hide/SKILL.md` | What the agent does: ground rules and five tasks (install, hide, keep outputs private, check, unhide/remove). |
| `skills/skill-hide/references/commands.md` | The exact git commands for each step, in bash and PowerShell. |
| `skills/skill-hide/scripts/pre-commit` | Optional git hook: stops commits that add hidden files. |
| `README.md` | For humans: install, example prompts, how it works. |

## Decisions already made (don't undo without asking)

- **A skill, not a tool** (since Nov 2026). The old Node.js CLI could never tell skill output from product code; the agent can. Keep it a skill.
- **Hide with `.git/info/exclude`**, inside one managed block with `# group:` lines. Never edit `.gitignore`.
- **Skill outputs go to `.personal/`.** A note appended to each hidden skill's `SKILL.md` tells any agent to do that; `CLAUDE.local.md` carries a backup rule for Claude Code.
- **`skills-lock.json`** goes back to the team's version after installs.
- **No skip-worktree, ever.** It made `git pull` stop with conflicts.
- **Safety:** the agent never commits, pushes, resets, cleans or deletes without a yes. It removes links by themselves (Windows junctions with `cmd /c rmdir`), and never undoes the user's own changes to team files.
- **Keep it simple.** Prefer telling the user plainly what to do over clever automation.

## Working rules

- Keep `SKILL.md` short and imperative, and put exact commands in `references/commands.md`.
- Test changes by having a fresh agent follow the skill in a throwaway git repo with a real `npx skills add`, then check `git status`, the exclude block and the lock file.
- The hook script must stay POSIX `sh`, and must never block on anything but files from `.git/info/exclude`.
