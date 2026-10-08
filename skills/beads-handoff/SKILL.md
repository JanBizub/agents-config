---
name: beads-handoff
description: "Decide whether to continue the current beads session or hand off to a fresh one. Measures real context usage, closes out beads state (statuses, bd remember, git), and produces a paste-ready prompt for the next session window."
argument-hint: "Optional: the bead(s) the next session should work on (e.g. 'paw.6 next' or 'continue the frontend beads')"
user-invocable: true
---

# Beads Session Handoff

## Purpose

Decide — with a real measurement, not a guess — whether to keep working in the current beads
session or start a fresh one, and make the transition lossless when a handoff is the right
call.

This is the companion to `beads-operator`. That skill runs the work; this one ends it cleanly.
A handoff is only worth doing if the next session can claim the next bead without re-deriving
anything, so **persisting beads state is the mandatory part** and the paste block is the
deliverable.

## When To Use

- The user asks to hand off, or asks "should I start a new session?"
- A bead or a group of beads was just closed and the next bead is in a different layer.
- You notice you are re-reading files you already read earlier in the session.
- `bd ready` shows the next work is unrelated to what is currently loaded in context.

## Procedure

### 1. Measure the real context usage

Do not estimate. The local session store records the exact prompt size of every model call,
and the newest row is the current context occupancy.

Take the session id from the session folder path given in the session context
(`.../session-state/<session-id>`), then run this through the `session_store_sql` tool with
`source: "local"`:

```sql
SELECT MAX(input_tokens) AS context_tokens
FROM assistant_usage_events
WHERE session_id = '<session-id>'
```

Convert to a percentage of the model's context window. Default-tier windows are commonly about
200k tokens; a `long_context` tier is larger. State which window you assumed.

If the query returns nothing (brand-new session, or usage not yet flushed), say the number is
unavailable and fall back to the qualitative signals alone. Never invent a figure.

### 2. Read the beads-specific signals

Tokens alone do not decide this. Run `bd ready` and `bd list --status=in_progress`, then weigh:

**Pushes toward handoff**
- The next ready bead touches a different layer than the context holds — for example the
  current session is full of backend F# and the next bead is frontend React, or vice versa.
- A bead was just closed: a clean boundary where nothing is half-built.
- Context is dominated by artifacts irrelevant to the next bead — planning dumps, generated
  scripts, large files read once for design work.
- You have re-read the same file more than once because the earlier read fell out of context.

**Pushes toward staying**
- A bead is `in_progress` and mid-edit, mid-debug, or mid-test-loop.
- The next bead reuses the same reference files already loaded — sibling beads in a chain
  usually do.
- Only a small, well-understood remainder is left on the current bead.

### 3. Decide

| Context used | Default recommendation |
|---|---|
| Under 40% | Keep the session. Do not hand off without a strong qualitative reason. |
| 40–65% | Keep going, but name the bead boundary where a handoff will make sense. |
| 65–80% | Hand off — unless a bead is mid-edit or mid-debug, in which case finish that first. |
| Over 80% | Hand off. Finish only what cannot be safely interrupted. |

Beads signals can move the recommendation one band, never two. Say explicitly when they do.

Never hand off with a bead left `in_progress` and its working tree half-edited. Finish the
edit, or revert it and put the bead back to open with a note explaining why.

### 4a. If keeping the session

Answer briefly: the measured number, the recommendation, and the bead at which a handoff will
make sense. Do not produce a report. Then continue the work.

### 4b. If handing off — close out beads state first

**Mandatory, and it comes before writing the report.** Context is about to be discarded;
anything only in your head is lost. This is the `beads-operator` session close protocol, made
explicit:

1. **File issues for remaining work** — `bd create` anything discovered but not done. Do not
   leave follow-ups in prose.
2. **Update bead statuses** — `bd close` what is finished. For anything still `in_progress`,
   update the description or notes so it states exactly where it stands and what comes next.
   A stale `in_progress` bead with no notes is the most common way a handoff loses work.
3. **`bd remember`** — store decisions, constraints and gotchas learned this session that are
   not obvious from the code. `bd prime` surfaces these automatically in the next session,
   which makes it the most reliable channel available. Use a stable `--key` so a later session
   updates it in place rather than duplicating it.
4. **Plan or notes file** — update it if one exists, and reference it by absolute path: the
   session folder is per-session and the next session's folder will differ.
5. **Working tree** — run `git status` and report exactly what is uncommitted. Follow the
   active agent profile: conservative and minimal report and wait; do not commit, push, or run
   `bd dolt push` unless explicitly authorised. If a required sync is blocked, report the exact
   command and error.

Verify each step actually happened before continuing. A handoff report pointing at state that
was never written is worse than no handoff at all.

### 5. Write the handoff report

Two parts: a short status summary for the user, then the paste block.

The paste block must be a single fenced code block the user can copy into a new session window
unchanged, and it must be self-contained — assume the next session knows nothing beyond what
`bd prime` gives it.

Include, in this order:

1. **Skills to invoke** — always `/beads-operator`, plus any domain skill the next bead needs
   (for example `/add-mcp-tool`, `/add-aggregate-auth-tests`, `/config-migration-compatibility`,
   `/verify-generated-sql-test`). Add any `/add-folder` or workspace setup the session needs.
2. **The objective** — the bead ids to work, in order, in one or two sentences.
3. **What to read first** — `bd show <id>` for each target bead, absolute paths to plan or
   notes files, and the `bd remember` keys that matter.
4. **Reference files with line ranges** — the files this session found useful, so the next one
   does not search for them again. Line ranges matter for large files.
5. **Constraints** — branch name, commit and push policy, anything that must not change.
6. **Known traps** — mistakes already made or narrowly avoided, and related bead ids that
   document them.

Keep it tight. It is a starting prompt, not a transcript. Prefer pointing at bead descriptions
over restating them — the beads are the source of truth.

## Report Template

```
## Handoff

**Context:** <N>k / <W>k tokens (<P>%) — <recommendation>
**Reason:** <one line>

**Beads:** closed <ids> · created <ids> · in progress <ids + where they stand>
**Memory:** <bd remember keys written or updated>
**Working tree:** <branch> — <uncommitted summary, or "clean">
**Loose ends:** <anything the next session must decide, or "none">
```

Followed by the paste block:

````
```
/beads-operator
<other skills>

<objective — which beads, in what order>

Read first:
- bd show <id> (and <ids>)
- <absolute path to plan/notes>
- memory <key> (bd prime shows it)

Reference files:
- <path>:<lines> — <why>

Branch: <branch>. <commit/push policy>.
Traps: <known pitfalls, with bead ids>
```
````

## Rules

- **Measure, never guess.** Always run the usage query. If it fails, say the number is
  unavailable.
- **Persist before reporting.** The report is valid only once the beads state behind it exists.
- **Never hand off mid-edit.** Leave a coherent tree and a bead whose status matches reality.
- **Never commit, push, or `bd dolt push`** as part of a handoff unless explicitly authorised.
  Report the proposed commands instead.
- **Prefer beads and `bd remember` over prose.** They survive; a report in a closed terminal
  does not.
- **Be honest about cost.** If a handoff means re-reading several large files, say so, so the
  user can weigh it.
- **Recommend staying when staying is right.** This skill exists to make a decision, not to
  justify a handoff.
