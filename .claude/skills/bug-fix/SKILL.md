---
name: bug-fix
description: Takes one bug all the way to a pull request by running the bug-fixer subagent, then the fix-reviewer subagent (up to 2 review rounds), then committing, pushing a new branch and opening a pull request on the user's own repository. Use it when the user types /bug-fix, asks to "fix this bug and open a PR", or wants to act on a GO from /bug-triage. Input is a bug description (plain English, a Notion ticket link or a GitHub issue link) or nothing, in which case it uses the /bug-triage reports already in the conversation. It runs without asking any question, never merges, never pushes to the default branch, and never edits code itself. It answers with a fixed two- or three-line result.
argument-hint: [bug description, Notion ticket link or GitHub issue link]
---

# /bug-fix

You are a **manager**. Your single mission: take one bug to a pull request by running the two expert subagents in order (**bug-fixer**, then **fix-reviewer**), then committing, pushing and opening the pull request. You do none of the experts' work yourself.

Write in plain language. Every path is written from the project root.

## Input

The input is: $ARGUMENTS

- **Nothing:** you were called in the session where /bug-triage just ran. Take the description of the bug from the conversation: the latest bug-reproducer and bug-analyst reports, plus the original bug. Also look there for a Slack channel, a thread timestamp, and the link of a Notion ticket.
- **A Notion ticket link, a GitHub issue link, or a few sentences:** that is the description of the bug. Pass it on as is; bug-fixer opens links itself. Also note any Slack channel, thread timestamp or Notion link in it.

If you find no description of a bug at all, answer with this one line and stop:

`Stopped: no bug description was given or found in the conversation.`

## Hard rules

- **Never ask the user anything.** Run from start to finish on your own.
- **Never merge the pull request, and never push to the default branch.** Merging is the user's decision. Never force-push.
- **Never edit the code or the tests yourself**, and never read the code to judge the fix: that is the experts' job.
- **Never rewrite or summarize a report.** Pass reports between subagents, and into the pull request, exactly as returned.
- **If a step fails, stop.** If a subagent is missing from `.claude/agents/`, errors out, returns nothing, or returns a report whose first line isn't the expected one, or if a git or gh command fails with no fallback below, answer with one line saying what failed and why, and stop. Go back to the starting branch first (step 7).

## Steps, in this order

### 0. Check that the experts exist
Check that `.claude/agents/bug-fixer.md` and `.claude/agents/fix-reviewer.md` both exist. If one is missing, answer `Stopped: the <name> subagent does not exist in .claude/agents/.` and stop.

### 1. Gather the description
Collect the description of the bug as described in Input, word for word. Note separately, if present: the **Slack channel**, the **thread timestamp**, the **Notion ticket link**, and the **bug link** (GitHub issue or Notion).

### 2. Create a branch
- Note the current branch (`git branch --show-current`): this is the **starting branch**.
- Find your repository and its default branch from the **origin** remote only:
  - `git remote get-url origin` gives `OWNER/REPO`.
  - `gh repo view OWNER/REPO --json defaultBranchRef --jq .defaultBranchRef.name` gives the **default branch**.
- Create and switch to a new branch named after the bug, in short lowercase words joined by dashes, starting with `fix/`, for example `fix/diamond-draws-ellipse`. **In a Claude Code cloud session, start the name with `claude/`** instead (for example `claude/fix-diamond-draws-ellipse`), so that the session is allowed to push it. If the name already exists, add `-2`, `-3`…
- Uncommitted files that were already there stay as they are: you will commit only the files of the fix.

### 3. Call bug-fixer
Call the **bug-fixer** subagent (`subagent_type: "bug-fixer"`) with the description, copied as is. Its first line must start with `**Fixed:`.
- `**Fixed: no**`: go straight to the answer (step 6). No pull request.
- `**Fixed: yes**`: go on.

### 4. Call fix-reviewer (at most 2 rounds)
Call the **fix-reviewer** subagent (`subagent_type: "fix-reviewer"`) with two things, clearly labelled: `Bug description:` (copied as is) and `bug-fixer report:` (copied as is). Its first line must start with `**Review:`.
- `**Review: approved**`: go to step 5.
- `**Review: changes requested**`:
  - **Round 1:** call bug-fixer again with the description and, labelled `fix-reviewer requests:`, the full fix-reviewer report copied as is. If it now answers `**Fixed: no**`, go to the answer. Otherwise call fix-reviewer again with the description and the **new** bug-fixer report. This is round 2.
  - **Round 2:** if it is still not approved, go straight to the answer. No pull request.

### 5. Commit, push and open the pull request
- **Commit only the fix:** stage exactly the files on the "Files changed" line of the **last** bug-fixer report, including the test file. Stage them by name (`git add <file> <file>`), never `git add .` or `git add -A`. Check with `git diff --cached --name-only` that nothing else is staged.
- Commit with a message whose first line is `Fix: <what was fixed, in the user's words>`, and end it with:
  ```
  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
  ```
- Push the branch to **origin**: `git push -u origin <branch>`.
- Open the pull request **on OWNER/REPO (origin) toward its default branch**, never on another repository, even if GitHub suggests the original project the fork comes from:
  `gh pr create --repo OWNER/REPO --base <default branch> --head <branch> --title "<title>" --body-file <file>`
  Write the body to a temporary file outside the project first.
- **If `gh pr create` is refused,** open it with the REST API instead:
  `gh api repos/OWNER/REPO/pulls -f title="<title>" -f head="<branch>" -f base="<default branch>" -F body=@<file>`
- Keep the pull request's link.

### 6. Answer back
Use exactly one of the forms below, and nothing else.

### 7. Go back and notify
- Switch back to the **starting branch** (`git switch <starting branch>`). If no pull request was opened, delete the fix branch you created (`git branch -D <branch>`). It has no commits of its own and was never pushed.
- **Slack:** if you have a Slack channel and a thread timestamp, and a Slack connector is available (search for it with ToolSearch, keyword "slack send message"), post the same answer as a reply in that thread. If it isn't available, skip this silently.
- **Notion:** if you have a Notion ticket link, a pull request was opened, and a Notion connector is available, add the pull request link to that ticket (a line `Pull request: <link>` at the end of the page). Don't change anything else on the ticket. If it isn't available, skip this silently.

## The pull request

- **Title:** what was fixed, in one short sentence, in the user's words. For example: `Diamond tool draws a diamond again (was drawing a circle)`.
- **Description**, four short parts, with the reports copied, not rewritten:

```
## The bug
<the original bug, in the user's words> <link to the issue or ticket, if there is one>

## Steps to reproduce
<the "1. Steps to reproduce" section of the bug-reproducer report, copied as is; if there is none, the steps from the bug description, copied as is>

## The fix
<the last bug-fixer report, copied as is>

## Checks
<the last fix-reviewer report, copied as is>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Output

### When the pull request is open

```
Fixed: yes
Review: approved
Pull request: <link>
```

### When it is not

```
Fixed: no
Why: <copied from the "Where it's stuck" block of the last bug-fixer report>
```

or, after 2 review rounds:

```
Review: changes requested
Why: <copied from the "2. Changes requested" list of the last fix-reviewer report, on one line, items separated by "; ">
```

### When a step fails

One line only, for example:

```
Stopped: git push failed: permission denied to OWNER/REPO.
```
