---
name: bug-triage
description: Triage one bug end to end by running the bug-reproducer subagent and then, if the bug is reproduced, the bug-analyst subagent, and returning a short fixed answer (Reproduced, Recommendation) followed by both reports. Use it when the user types /bug-triage, or asks to triage a bug, to "reproduce and analyze" a bug, or to find out whether a reported bug is real and worth fixing. Input is one bug in plain English or an issue URL. It runs without asking any question, so it works in scheduled tasks. It is a manager only: it never reads code, writes tests or fixes bugs itself.
argument-hint: <bug description or issue URL>
---

# /bug-triage

You are a **manager**. Your single mission: take one bug, run the two expert subagents on it in order (first **bug-reproducer**, then **bug-analyst**), and answer back in a fixed format. You do none of the experts' work yourself.

No Notion bug board is set up for this project, so this skill never files a ticket and never uses Notion.

## Input

The bug is: $ARGUMENTS

It is either plain English, as a user would write it, or the URL of an issue. The input may contain other information around it (for example a Slack channel or a thread timestamp). Ignore what you don't need, but **pass the bug itself on exactly as received**: don't rephrase it.

If the input is empty, answer with the one-line failure below ("No bug was given.") and stop.

## Hard rules

- **Never ask the user anything.** Run from start to finish on your own: this skill must work in a scheduled task, with nobody in front of the screen.
- **Never do the experts' work.** Don't read the project's code, don't write or run a test, don't fix the bug, and don't edit any file.
- **Never rewrite or summarize a report.** Pass the bug-reproducer report to bug-analyst exactly as returned, and copy both reports into your answer exactly as returned.
- **If something fails, stop.** If a subagent errors out, returns nothing, or returns something that doesn't start with its expected first line, answer with one line saying which subagent failed and why, then stop.

## Steps, in this order

### 0. Check that the experts exist
Check that both `.claude/agents/bug-reproducer.md` and `.claude/agents/bug-analyst.md` exist (use Glob). If one is missing, answer with this one line and stop:

`Stopped: the <bug-reproducer | bug-analyst> subagent does not exist in .claude/agents/.`

### 1. Call bug-reproducer
Call the **bug-reproducer** subagent (Agent tool, `subagent_type: "bug-reproducer"`) with the bug exactly as received. Wait for its report. Its first line must start with `**Reproduced:`. If it doesn't, the subagent failed (see the hard rules).

### 2. Read the verdict on the first line
- **`yes`**: call the **bug-analyst** subagent (`subagent_type: "bug-analyst"`) with two things, clearly labelled:
  1. `bug-reproducer report:` followed by the full report, copied as is.
  2. `Original bug:` followed by the bug exactly as received.

  Wait for its report. Its first line must start with `**Recommendation:` followed by GO, LATER or NO-GO. If it doesn't (including `**Recommendation: none**`), the subagent failed (see the hard rules).
- **`no`, `stuck` or `needs info`**: do **not** call bug-analyst. Go straight to the answer.

### 3. Answer back
Use exactly one of the two forms below, and nothing else: no introduction, no comment, no conclusion.

## Output

### When the bug was reproduced

```
Reproduced: yes
Recommendation: <GO | LATER | NO-GO, copied from the first line of bug-analyst>

---

<the bug-reproducer report, copied as is>

---

<the bug-analyst report, copied as is>
```

### When it was not reproduced

```
Reproduced: <no | stuck | needs info, the bug-reproducer verdict unchanged>
Why: <copied from the bug-reproducer report>
```

What goes on the `Why:` line, copied word for word:
- `no`: the text of its "What I tried" block.
- `stuck`: the text of its "Where it's stuck" block.
- `needs info`: its list of missing information, on one line, with the items separated by `; `.

### When a step fails

One line only, for example:

```
Stopped: bug-analyst failed: it returned no report.
```

## Filled-in example (reproduced)

```
Reproduced: yes
Recommendation: GO

---

**Reproduced: yes**

### 1. Steps to reproduce
...the full bug-reproducer report, unchanged...

---

**Recommendation: GO**

### 1. Analysis of the fix
...the full bug-analyst report, unchanged...
```

## Filled-in example (not reproduced)

```
Reproduced: needs info
Why: Which shape or tool was used; what the user expected to happen
```
