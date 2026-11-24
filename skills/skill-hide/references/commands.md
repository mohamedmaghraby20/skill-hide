# skill-hide: exact commands

Run everything from the repo root. Each step shows **bash** (macOS, Linux, Git Bash; not plain `sh`) and **PowerShell** (Windows) when they differ. `<path>` is always a repo path with forward slashes, like `.agents/skills/tdd`.

## Where things are

```bash
cd "$(git rev-parse --show-toplevel)"
G="$(git rev-parse --git-common-dir)"          # shared .git folder (worktrees too)
EX="$G/info/exclude"                           # the private ignore list
```

```powershell
Set-Location (git rev-parse --show-toplevel)
$G = git rev-parse --git-common-dir
$EX = "$G/info/exclude"
```

Edit `$EX` with your file-editing tool. Change only the lines between the two skill-hide markers. If the file or the block doesn't exist, create it (see the SKILL.md example).

**Never edit the block with `sed`, `echo >>`, PowerShell string replace or an ad-hoc script.** The `# group:` line contains `|`, quotes, `*` and `--` flags, which shells and regexes mangle. Use your file-editing tool, and replace whole lines:

- **Add paths to a group:** insert new lines right after the group's last path line.
- **Rewrite a group line** (e.g. a new install command): replace that one line, keeping the form `# group: <name> | installed with: <command>`.
- **Remove a group:** delete its `# group:` line and every path line below it, up to the next `# group:` line or the end marker.

Afterwards, print the block and check that a path really is hidden:

```bash
sed -n '/>>> skill-hide/,/<<< skill-hide/p' "$EX"
git check-ignore -v --no-index -- .agents/skills/<name>     # must print a line from info/exclude
```

```powershell
$b = Get-Content $EX; $s = [array]::IndexOf($b, ($b -match '>>> skill-hide')[0]); $e = [array]::IndexOf($b, ($b -match '<<< skill-hide')[0]); $b[$s..$e]
git check-ignore -v --no-index -- .agents/skills/<name>
```

## Before and after pictures (Task 1, steps 2 and 4)

```bash
git status --porcelain=v1 --untracked-files=all > "$G/sp-before.txt"
git ls-files --error-unmatch skills-lock.json >/dev/null 2>&1 && echo "lock is committed"
git diff --quiet HEAD -- skills-lock.json CLAUDE.md AGENTS.md || echo "team files already changed: don't undo those"
# … run the install …
git status --porcelain=v1 --untracked-files=all > "$G/sp-after.txt"
comm -13 <(sort "$G/sp-before.txt") <(sort "$G/sp-after.txt")   # what's new or newly changed
rm "$G/sp-before.txt" "$G/sp-after.txt"
```

```powershell
git status --porcelain=v1 --untracked-files=all | Set-Content "$G/sp-before.txt"
git ls-files --error-unmatch skills-lock.json 2>$null; if ($?) { "lock is committed" }
git diff --quiet HEAD -- skills-lock.json CLAUDE.md AGENTS.md; if (-not $?) { "team files already changed: don't undo those" }
# … run the install …
git status --porcelain=v1 --untracked-files=all | Set-Content "$G/sp-after.txt"
Compare-Object (Get-Content "$G/sp-before.txt") (Get-Content "$G/sp-after.txt") | Where-Object SideIndicator -eq '=>' | ForEach-Object InputObject
Remove-Item "$G/sp-before.txt", "$G/sp-after.txt"
```

How to read the lines: `?? path` is a new file. ` M path` or `M  path` is a changed committed file. On Windows, git may list the files *inside* a link (like `.claude/skills/tdd/SKILL.md`), because it can be a junction or a symlink. Either way, hide the link itself: `.claude/skills/tdd`.

## Safe to hide?

```bash
git ls-files -- <path>                  # must print nothing
```

After editing the block, this must print nothing:

```bash
git status --porcelain --untracked-files=all -- <path> <path2>
```

## Lock file (Task 1, step 7, and Task 4, step 4)

Put back the team's version (only if it had no changes before the install):

```bash
git checkout HEAD -- skills-lock.json
```

If the user has other changes in the lock file of their own, remove only the hidden skills' entries instead. This needs Node, which `npx skills` already requires:

```bash
node -e "const f='skills-lock.json',fs=require('fs'),j=JSON.parse(fs.readFileSync(f,'utf8'));for(const n of process.argv.slice(1))delete j.skills[n];fs.writeFileSync(f,JSON.stringify(j,null,2)+'\n')" skill-a skill-b
```

Which hidden skills are in the lock file but not in the team's version:

```bash
git diff HEAD -- skills-lock.json
```

## Team files changed by the install (Task 1, step 8)

```bash
git diff HEAD -- CLAUDE.md               # show the added lines
# copy the added lines into CLAUDE.local.md with your editing tool, then:
git checkout HEAD -- CLAUDE.md           # back to the team's version
```

## Links: find the real skill file (Task 3)

```bash
readlink -f .claude/skills/<name>        # where the link points
```

```powershell
(Get-Item .claude\skills\<name>).Target
```

If both `.agents/skills/<name>` and `.claude/skills/<name>` exist and one is a link to the other, edit only the real one.

## The personal note (Task 3, step 2)

Append this to the end of each hidden skill's `SKILL.md`, once. Skip it if the marker line is already there.

```markdown

<!-- skill-hide: personal skill -->
## Personal install (added by skill-hide)

This skill is installed privately in this repo. The user's teammates must not see anything it creates.

- Any file this skill tells you to create or update (plans, specs, ADRs, ideas, notes, todo lists, checklists, constraint files, reports) goes under `.personal/`, at the same relative path: a file the skill names as `<folder>/<file>.md` goes to `.personal/<folder>/<file>.md`. Read those files from there too.
- Product code, tests and config that the user asked for go in their normal place.
- If this skill tells you to add lines to `AGENTS.md`, `CLAUDE.md` or another committed file, put them in `CLAUDE.local.md` instead, and tell the user.
```

## The CLAUDE.local.md section (Task 3, step 3)

Keep one section like this. Update the list of skills when groups change.

```markdown
<!-- >>> skill-hide >>> -->
## Personal skills (skill-hide)

These skills are installed privately: <skill-a>, <skill-b>. The user's own global (user-level) skills are personal too.
Files these skills tell you to create go under `.personal/` at the same path (`<folder>/<file>.md` → `.personal/<folder>/<file>.md`). Lines they want in `AGENTS.md` or `CLAUDE.md` go here instead, below this paragraph. Never commit `.personal/` or the skill folders.
<!-- <<< skill-hide <<< -->
```

## Checking before a commit (Task 4)

Staged files that are hidden (step 1):

```bash
git diff --cached --name-only --diff-filter=A | git check-ignore --no-index --stdin
git restore --staged -- <file>          # for each one
```

New files, untracked and staged (step 2):

```bash
git status --porcelain --untracked-files=all | grep -E '^(\?\?|A )'
```

Does a hidden skill mention this file or its folder? (The personal note uses only placeholders, so it never matches.)

```bash
grep -rlF "tasks/plan.md" .agents/skills/<hidden-skill>
grep -rlF "tasks/" .agents/skills/<hidden-skill>
```

```powershell
Get-ChildItem -Recurse -File .agents\skills\<hidden-skill> | Select-String -SimpleMatch -List -Pattern "tasks/plan.md"
```

Also check the user's global skills folders (for example `~/.claude/skills`, `~/.agents/skills`). Files from those skills are personal too.

Move a skill file into `.personal/`, same path (unstage it first if it's staged):

```bash
mkdir -p ".personal/$(dirname <path>)" && mv <path> ".personal/<path>"
```

```powershell
New-Item -ItemType Directory -Force (Split-Path ".personal/<path>") | Out-Null; Move-Item <path> ".personal/<path>"
```

Lines added to team files (step 3):

```bash
git diff HEAD -- AGENTS.md CLAUDE.md
```

Did the team add a hidden path? (step 7). `git fetch` talks to the remote but changes none of the user's files:

```bash
git fetch --quiet
git ls-tree -r --name-only @{u} -- <path> <path2>     # anything printed = the team has it
```

Committed by mistake, not pushed (step 8). Give these to the user; don't run them:

```bash
git rm -r --cached -- <path>
git commit -m "Stop tracking personal files"
```

## Deleting (Task 5, remove)

Always list first and get a yes.

```bash
rm <link>                 # a symlink: no trailing slash, removes only the link
rm -rf <folder>           # a real folder
```

```powershell
cmd /c rmdir "<link>"                     # a junction or folder link: removes only the link
Remove-Item -Recurse -Force <folder>      # a real folder (never use this on a junction)
```

Check what something is first: `ls -la <path>` (bash), or `(Get-Item <path>).LinkType` (PowerShell: `Junction` or `SymbolicLink` means a link).

## The git hook (optional)

```bash
H="$G/hooks/pre-commit"
git config --get core.hooksPath && echo "STOP: shared hooks folder, don't install"
```

- **No `$H` yet:** copy this skill's `scripts/pre-commit` to `$H`, then `chmod +x "$H"` (not needed on Windows).
- **`$H` already exists:** append everything between the `# >>> skill-hide >>>` and `# <<< skill-hide <<<` lines of `scripts/pre-commit` to the end of it. If `$H` already has such a section, or an older `# >>> skill-pocket >>>` … `# <<< skill-pocket <<<` section, replace that section instead.

To remove it later, delete that section, and delete `$H` if nothing else is left but `#!/bin/sh`.
