# skill-hide

[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

An agent skill for using AI skills **privately in a team repo**. Install any skill the normal way, and your teammates never see it: not the skill's files, not the entry in `skills-lock.json`, and not the plans, specs and notes the skill writes while you use it.

Works with any agent that supports skills (Claude Code, Codex, Cursor, OpenCode…). Nothing to install in the project, nothing your team has to do.

## Install

Install it **globally**, so it isn't part of any project:

```bash
npx skills add mohamedmaghraby20/skill-hide -g
```

Or copy `skills/skill-hide` into your agent's personal skills folder (for example `~/.claude/skills/`).

## Use it

Just ask your agent:

| You say | What happens |
|---|---|
| "Install `addyosmani/agent-skills` for me only, with skill-hide" | The agent installs it, hides it, and keeps it out of `skills-lock.json`. |
| "Hide the skills I just installed from my team" | Same, for skills that are already there. |
| "Is it safe to commit?" | The agent checks that nothing personal is about to leak, and fixes it. |
| "Unhide tdd" / "Share tdd with the team" | The skill becomes visible again (and goes into the lock file, if you share it). |
| "Update my personal skills" | Reinstalls them from the saved command, keeps them hidden, puts the notes back. (Not `npx skills update`: it skips skills that aren't in the lock file.) |
| "Remove the addy skills" | Deletes them, after showing you the list. |
| "Add the skill-hide commit hook" | A small git check stops any commit that includes a hidden file. |

## How it works

- **Skill folders** are listed in `.git/info/exclude`, git's private ignore list. It works like `.gitignore`, but is never committed.
- **What the skills write** (`SPEC.md`, `tasks/plan.md`, ADRs, notes) goes into one hidden folder, `.personal/`. skill-hide adds a short note to each personal skill telling the agent to do that, so it works with any agent.
- **`skills-lock.json`** is put back to the team's version after each install. Your skills still work, because agents read skill folders, not the lock file.
- **Lines a skill wants in `CLAUDE.md` or `AGENTS.md`** go into `CLAUDE.local.md`, which is hidden too.
- **Before a commit**, ask "is it safe to commit?". The agent checks everything, including skill files written in the wrong place.
- **The optional git hook** works without an agent, but it only knows hidden paths. If an agent ignores the note and writes `tasks/todo.md` at the root, the hook lets it through; only the agent's check catches that.

skill-hide never commits, pushes, or edits `.gitignore`. It never deletes anything without asking first.

## Good to know

- If your team later commits the same skill, your next `git pull` replaces your copy with theirs. Ask "is it safe to commit?" before you pull, and the agent warns you.
- `git clean -fdx` deletes hidden files. The agent can tell you the command to reinstall them.
- Hiding works on every branch, not just the one you installed on. Your skills and `.personal/` stay in place, and your agent keeps using them, when you switch branches.
- Worktrees of one repo share the hidden list.
- Updating a skill replaces its files, so the note telling the agent to use `.personal/` is lost. Ask the agent to update your personal skills (it puts the note back), or ask "is it safe to commit?" afterwards.

## License

[MIT](LICENSE)
