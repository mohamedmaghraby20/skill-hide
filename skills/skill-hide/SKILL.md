---
name: skill-hide
description: Use AI agent skills privately in a shared team git repo. Installs or adopts skills and hides them, plus everything they write while you use them (plans, specs, ADRs, notes, todo lists), from git and teammates using .git/info/exclude; keeps your skills out of skills-lock.json; checks before commits that nothing personal leaks. Use when the user wants to install a skill "just for me", privately or hidden; hide, unhide, share or remove personal skills; asks whether it is safe to commit or push; or says "skill-hide".
---

# skill-hide

Personal skills in a team repo: the user gets the skill, the team sees nothing. Not the skill folders, not the lock file entry, not the files the skill writes later.

How it works, in one picture:

- **Skill folders** are listed in `.git/info/exclude`, git's private ignore list. It works like `.gitignore` but is never committed.
- **Files the skills write** (plans, specs, ADRs, notes) go into one hidden folder, `.personal/`. A short note added to each personal skill tells you (the agent) to do that.
- **`skills-lock.json`** goes back to the team's version after every install.
- **A check** before commits (by you, and optionally a git hook) catches anything that slipped.

Exact commands for every step are in [references/commands.md](references/commands.md). Read it before running any step below.

## Ground rules (never break these)

1. **Touch only these:** the managed block in `.git/info/exclude`; the `.personal/` folder; the skill-hide section of `CLAUDE.local.md`; the personal skills' own files; and changes the install itself made to team files (to undo them).
2. **Never** edit `.gitignore`, commit, push, pull, stash, reset, rebase, run `git clean`, or use `git update-index --skip-worktree`. Show the user the command instead.
3. **Never delete anything** without listing it and getting a clear yes. A link (or Windows junction) is removed by itself, never the folder it points to.
4. **Never hide a path that has committed files in it.** Check with `git ls-files -- <path>` first. If it prints anything, don't hide it.
5. **Never undo a change to a team file that existed before the install.** It's the user's own work. Tell them instead.
6. Work from the repo root (`git rev-parse --show-toplevel`). The exclude file is at `$(git rev-parse --git-common-dir)/info/exclude`, which is shared by all worktrees of the repo.
7. **End every task with a short report:** what you hid, moved or changed, and how to undo it.

## The managed block

Everything skill-hide hides is listed between two marker lines at the end of the exclude file. Lines starting with `# group:` name each install, and record its command so it can be reinstalled:

```
# >>> skill-hide: hidden files (managed by skill-hide, do not edit) >>>
/.personal
/CLAUDE.local.md
# group: addy | installed with: npx skills add addyosmani/agent-skills --skill '*'
/.agents/skills/idea-refine
/.claude/skills/idea-refine
# <<< skill-hide: hidden files <<<
```

- One path per line, starting with `/` (from the repo root). No trailing slash, so links and folders both match. Escape `*`, `?` and `[` with `\`.
- Create the block if it's missing. Never change lines outside it.
- If you find an older block with `skill-pocket` in its marker lines, it's this skill under its old name: rename the markers to `skill-hide`, keep its lines, and delete `.git/skill-pocket.json` if it exists.
- `/.personal` is always in the block. `/CLAUDE.local.md` is too, unless the team committed that file.

## Task 1: install a skill privately

The user gives an install command, or asks for a skill and you find the command (for example `npx skills add <owner>/<repo> --skill <name>`).

1. **Offer global first.** If the skill isn't specific to this project, say: "Install it for you in every project instead? (`-g`) Then there's nothing to hide." If the user wants it in this project, go on.
2. **Take a "before" picture** of `git status`, and note whether `skills-lock.json` is committed and whether team files like `CLAUDE.md` and `AGENTS.md` already have changes.
3. **Run the install command exactly as given.** Add the installer's own "yes" flag (`-y`) only if it would otherwise wait for input you can't give.
4. **Take an "after" picture.** New files are those in "after" but not in "before". Changed team files are modified files that weren't modified before.
5. **Choose the paths to hide:**
   - Inside any folder named `skills`, hide only the skill's own folder: `.agents/skills/tdd`, never `.agents` or `.agents/skills`.
   - Anywhere else, hide the highest new folder that has no committed files, or the single file.
   - Never hide something that existed before the install. It's the user's work.
6. **Add the paths to the block** under a `# group:` line. There's one group per install, named after the skill if there's one, otherwise after the pack (for example the repo name `agent-skills`). Then confirm `git status` no longer shows them.
7. **Put back `skills-lock.json`:**
   - If the team commits it and it was clean before, restore the team's version. The skills still work: agents read skill folders, not the lock file.
   - If it's a new file, hide it in the group instead.
8. **Handle the other changed team files:**
   - `CLAUDE.md` with only added lines: move those lines into the skill-hide section of `CLAUDE.local.md` (rewrite any file path in them to its `.personal/` path), then restore `CLAUDE.md`.
   - Any other team file (e.g. `AGENTS.md`): show the user the change and restore it after they say yes.
9. **Set up Task 3** for the new skills.
10. **Report.**

## Task 2: hide skills that are already installed

1. List what git can see that isn't committed (`git status --porcelain --untracked-files=all`).
2. Point out the skill folders, their links, and `skills-lock.json` changes. Ask which skills to hide, and the group name.
3. Do steps 5 to 10 of Task 1. For the lock file, remove only the entries of the skills being hidden, and only if the team's version doesn't have them (see the commands file).
4. Also look for files those skills already wrote (Task 4, step 2) and offer to move them into `.personal/`.

## Task 3: keep what the skills write private

Skills create files while they're used: `SPEC.md`, `tasks/plan.md`, `docs/adr/…`, `CONSTRAINTS.md`, notes. You (the agent) are the only one who knows whether a file came from a skill's instructions or is product code. So the rule travels with the skill itself, and every agent reads it when the skill is used.

1. **Make sure `.personal/` exists** and `/.personal` is in the block.
2. **Add the personal note** to the end of each hidden skill's `SKILL.md`, once (the marker line prevents duplicates). If `.claude/skills/<name>` is a link, edit the real file it points to. The exact text is in the commands file. In short, it says:
   - files this skill creates go under `.personal/`, at the same path (`tasks/plan.md` → `.personal/tasks/plan.md`), and are read from there too;
   - product code, tests and config go where they normally go;
   - lines this skill wants in `AGENTS.md`, `CLAUDE.md` or another committed file go in `CLAUDE.local.md` instead.
3. **Add the same rule, short, to `CLAUDE.local.md`,** in a marked skill-hide section listing the personal skills. The rule also covers the user's global (user-level) skills, which are personal too. This is a backup that Claude Code reads every session. Other agents rely on the note in step 2 and on Task 4.

Installers' update commands (e.g. `npx skills update`) replace the skill files and remove the note. Task 4 puts it back.

## Task 4: check before a commit or push

Do this when the user asks ("is it safe to commit?", "check skill-hide"), before you commit anything yourself in a repo that has the block, and after the user updates or reinstalls skills.

1. **Staged personal files:** a staged path that's in the block or under `.personal/` → unstage it.
2. **Skill files that leaked:** a new file (untracked or staged) that a hidden skill or a global skill names in its instructions (search their text for the file's path or its folder), or that is plainly skill output (a plan, spec, ADR, idea, todo list) → offer to move it to `.personal/` at the same path. Leave product code alone. When unsure, ask.
3. **Lines in team files:** added lines in `CLAUDE.md`, `AGENTS.md` and similar that mention a hidden skill, `.personal/`, or a skill-output file → offer to move them into the skill-hide section of `CLAUDE.local.md` (with paths rewritten to `.personal/`) and restore the team file. Only restore a team file if its other changes are also from skills; otherwise remove just those lines.
4. **Lock file:** if `skills-lock.json` now lists hidden skills that the team's version doesn't have, put those entries back the team's way.
5. **Personal notes:** re-add the Task 3 note to any hidden skill that lost it.
6. **Missing files:** a hidden path that no longer exists (e.g. after `git clean -fdx`) → tell the user, and give the install command from its `# group:` line.
7. **Team adopted a skill:** fetch, then check whether the upstream branch now has any hidden path. If it does, say: "Your team added `<skill>`. Your next pull replaces your copy with theirs; save your edits first." After the pull, remove those lines from the block.
8. **Already committed by mistake** (only on this computer, not pushed): give the user the commands to stop tracking the file. Don't run them.
9. **Report:** "Safe to commit", or what you fixed and what's left for the user.

**Optional git hook:** offer to install [scripts/pre-commit](scripts/pre-commit) (how to install it is in the commands file). It stops any commit that adds a hidden file, even when no agent is around. Don't install it when `core.hooksPath` is set; that folder may be shared with the team.

## Task 5: unhide, share, remove, uninstall

- **Unhide** (a group, or some skills of it): remove their lines from the block. The files show in `git status` again.
- **Share with the team:** unhide, remove the personal note from the skill's `SKILL.md`, and run the install again normally so the lock file gets the entry. Then the user commits.
- **Remove** (a group, or some skills of it): list what will be deleted, get a yes, then delete the files (links by themselves), remove the lines from the block, and remove their entries from the `CLAUDE.local.md` section. Never delete committed files.
- **Uninstall skill-hide from this repo:** remove the block, the `CLAUDE.local.md` section and the hook section. Ask before deleting `.personal/`; keep it by default.

## Good to know

- **Worktrees share one exclude file.** A skill hidden in one worktree may show as missing in another. That's normal.
- **`git clean -fdx` deletes hidden files.** Plain `git clean -fd` keeps them.
- **The team's `.gitignore` wins over the exclude file.** A `!` line there can un-hide a path. If `git status` shows a hidden path, run `git check-ignore -v --no-index <path>` and tell the user what you find.
- **Cloud agents** run on another machine and don't have the user's personal skills.
