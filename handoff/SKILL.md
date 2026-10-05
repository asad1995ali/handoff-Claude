---
name: handoff
description: Wrap up a coding session in any git-backed project — commit and push all pending changes, then update that project's handoff docs at the repo root (HANDOFF.md for current state, HANDOFF-history.md for the dated session log) so the next chat/session, which starts with zero memory of this one, has full context. Works in any directory/repo, not tied to one project. Use whenever the user says things like "wrap up", "let's push this", "commit and push", "deploy this", "end of session", "update the handoff doc", "let's call it there", or similar — in whatever project the chat is currently working in.
---

# Handoff: commit, push, and update the handoff docs

Every project loses conversation memory between chats. The handoff docs at
the repo root are what the *next* session reads to know what happened — treat
writing them as seriously as writing the code. A future Claude with zero
context about this conversation has to be able to pick up from them alone.

This skill is intentionally generic — it doesn't assume a stack, a build
tool, or a remote host. Discover the project's actual conventions each time
rather than assuming they match a previous project.

## The two files, and what goes in each

A single ever-growing `HANDOFF.md` eventually gets too big to read at session
start (hundreds of KB, tens of thousands of tokens). So the handoff is split
in two files, both at the repo root:

| | `HANDOFF.md` — current state | `HANDOFF-history.md` — session log |
|---|---|---|
| **Purpose** | What a new session needs to know *now* | What happened, session by session, and why |
| **Read when** | **Fully, at the start of every session** | **Never in full.** Grep it for a specific file/feature/table/session, then read only the matching sections |
| **Contains** | Tracking note; pointer section to the history file; what the project is; stack & environment; workflows (git, deploy, migrations, verification); how each page/feature/module works *today*; gotchas & hard-won lessons; known follow-ups / TODO; open multi-session work (status + next steps) | Dated `## Changes — session <date>` sections: what changed, which files/functions, why, rejected or reverted approaches, migrations, debugging detail |
| **How it's edited** | Kept current: edit sections in place when a fact changes, delete what's no longer true | Append-only: new sections go at the bottom, oldest first; old sections are never rewritten |
| **Size** | Small — aim for a few hundred lines | Grows without limit, which is fine since nobody reads it whole |

`HANDOFF.md` is the source of truth where the two disagree — the history
contains things that were later changed or reverted.

**The rule that makes this work:** anything a future session needs to know
*by default* must be in `HANDOFF.md`, not only in the history. If this
session changed how a feature works, its dated section goes in the history,
**and** a short current-state description goes into (or replaces the old one
in) the relevant section of `HANDOFF.md`, with a pointer like "details:
history, session <date>".

## 0. Pre-flight: confirm there's something to commit and somewhere to push it

Don't guess your way past a missing prerequisite — each of these is a real
decision point, not housekeeping, so surface it to the user instead of
picking a default silently.

**No project directory attached.** If the chat has no working directory yet
(brand-new session, nothing opened), or the working directory doesn't exist /
is empty with no sign of a project, stop and ask the user for the local
project folder before doing anything else. Don't invent a path.

**No git repo found.** Try `git rev-parse --show-toplevel` from the working
directory. If that fails, look for exactly one subdirectory containing a
`.git` folder and treat that as the repo root (this project's working
directory may be a parent folder wrapping the actual repo). If there are
multiple candidates, ask which one. If there are none at all, ask the user:
is there an existing repo this should be pointed at, or should one be
initialized here (`git init`)? Don't run `git init` unilaterally — repo
creation is the user's call, not a default to fall back on.

**No GitHub remote connected.** Once you have a repo root, check
`git remote -v` and whether the current branch has a tracked upstream
(`git rev-parse --abbrev-ref --symbolic-full-name @{u}`). If there's no
remote, ask the user whether to add one now (they'll need to give you the
GitHub repo URL, or say if they want a new repo created — GitHub repo
creation itself is also the user's call, don't do it unprompted) or to skip
pushing for this session. If they choose to skip, still do steps 2-5
(commit locally, update the handoff docs) and say clearly in the final
summary that changes were committed but **not pushed**, since there's nowhere
to push to.

**Handoff docs missing.** No "should I create them?" ask needed — a missing
`HANDOFF.md` just means this is the first handoff for this project. Create
both files fresh (see step 5). It does need **one** question the first time
only: whether the handoff docs should be committed to git (pushed like the
rest of the repo) or kept local-only (never staged, purely for this machine).
The answer covers both files. See step 5 for how to ask and where the answer
is recorded — check for that record on every future invocation instead of
asking again.

**Old single-file layout.** If `HANDOFF.md` exists but there's no
`HANDOFF-history.md` and `HANDOFF.md` still holds dated session sections,
the project predates the split. Ask the user once whether to split it now
(recommended once the file is more than a few hundred lines). If yes, do the
migration in step 5 before writing this session's entry. If no, record that
in the tracking note ("single-file layout, kept by choice (decided <date>)")
and fall back to inserting the dated section into `HANDOFF.md` as before.

## 1. Find the repo root

(Resolved in step 0 above — this step is just a reminder of the mechanics.)
The chat's working directory is not always the git repo itself — it may be a
parent folder that contains the actual project in a subfolder (or the repo
root itself).
- On Windows, prefer the PowerShell tool over Bash for git/npm commands —
  some environments have had Bash silently return no output for commands
  that work fine, so if Bash produces nothing, don't assume the command ran;
  switch to PowerShell.

## 2. See what actually changed

Run `git status` and `git diff` (and `git log -1` for context on the last
commit) from the repo root. Cross-reference the diff against what happened
earlier in *this conversation* — the diff shows *what* changed, the
conversation tells you *why*, which matters for the write-up in step 5.

If the tree is already clean, there's nothing to commit — only continue to
step 5 if the user still wants a handoff note (e.g. a session that was pure
investigation, or made changes through an external system like a database
console that won't show up as a file diff).

## 3. Sanity-check before shipping (best-effort, don't invent steps)

Look for a real verification step already established in this project before
pushing — check `package.json` scripts (`build`, `typecheck`, `test`,
`lint`), a `Makefile`, or notes in `CLAUDE.md`/`HANDOFF.md` about how this
project verifies itself. Run whatever's actually there. If nothing like that
exists, don't invent a build system — just proceed, and mention to the user
that no verification step was found.

## 4. Stage and commit

- Prefer targeted `git add <files>` over `git add -A`/`.` so you don't sweep
  up build artifacts, local env files, or logs that happen to be untracked —
  check `.gitignore` and use judgment about what's actually part of this
  session's work. (Some projects explicitly forbid `-A` in their
  `CLAUDE.md`/`HANDOFF.md` — follow that if stated.)
- Write the commit message to a temp file first (e.g. via the Write tool to
  `.git/COMMIT_MSG.txt`) and commit with `git commit -F <file>`, rather than
  `git commit -m "..."` — multi-line messages and special characters are
  unreliable through PowerShell/shell quoting.
- One commit for the session's work is usually fine. The handoff-doc update
  (step 6) can ride in the same commit or a trailing one — default to
  folding it in unless the session had several unrelated chunks of work
  worth separating in history. **Only fold the handoff docs in if the
  tracking note (see step 5) says they're committed to git** — if they're
  marked local-only, never `git add` either file, regardless of what else is
  being committed this session.

## 5. Write the handoff docs

### First handoff for a project (neither file exists)

Before writing, ask the user one question: should the handoff docs be
committed to git and pushed like the rest of the repo, or kept
**local-only** (never staged/committed, purely notes for this machine)?
Record the answer near the top of `HANDOFF.md`:

```markdown
## Handoff tracking
HANDOFF.md and HANDOFF-history.md are kept **local-only** — not committed to git. (decided <date>)
```
or
```markdown
## Handoff tracking
HANDOFF.md and HANDOFF-history.md are **committed and pushed** to git along with the rest of the repo. (decided <date>)
```

This line makes the decision durable — on every future invocation, read it
and follow it instead of asking again. Only re-ask if the user brings it up
themselves or the note is missing/ambiguous.

Then create:
- **`HANDOFF.md`**: the tracking note, the pointer section (template below),
  then a short description of the project, stack, workflow notes, and how
  the parts touched this session work.
- **`HANDOFF-history.md`**: a one-paragraph header (what the file is; read
  `HANDOFF.md` first; append-only, oldest first), then this session's dated
  section.

Pointer section template for `HANDOFF.md`, placed right after the tracking
note:

```markdown
## Session history lives in a separate file — don't read it up front
**Location:** `HANDOFF-history.md` at the repo root (<absolute path>).
**What's in it:** every dated "Changes — session" log, oldest first.
**Why it's not read at session start:** it grows every session and would eat
the context window on mostly-irrelevant history; parts of it are stale.
This file is the current-state reference and wins where the two disagree.
**When to read it (targeted, never whole):** before changing a
feature/file/table, grep the history for its name and read only the matching
sections (the why, rejected approaches, gotchas); when this file points at a
specific session; when the user asks why/when something was done.
**Writing handoffs:** append new dated sections to the bottom of the history
file; only edit this file when a current-state fact changes.
```

### Migrating an old single-file `HANDOFF.md` (if the user agreed in step 0)

Move every dated `## Changes — session …` section, verbatim and in order,
into a new `HANDOFF-history.md` (with the header paragraph above). Keep
everything else in `HANDOFF.md`: tracking note, project/stack/workflow
sections, gotchas, known follow-ups, open-work status. Add the pointer
section. Do it by line ranges with a script rather than re-typing content,
and check afterwards that the number of dated sections in the history
matches what was in the original. Then skim the most recent few dated
sections for current-state facts that never made it into the top sections,
and fold a short summary of each into `HANDOFF.md`.

### Every handoff (files exist)

Read `HANDOFF.md` fully. **Don't read `HANDOFF-history.md` in full** — to
pick the right section title, read only its last ~30 lines.

**1. Append the session log to `HANDOFF-history.md`.** Add a new section at
the very bottom titled `## Changes — session <date>` (use today's actual
date; if a section for today already exists, suffix `(part N)`):

```markdown
## Changes — session <date>

### <Feature or area name>
What changed, which files/functions, and any non-obvious reasoning.
```

Match the voice already in the file — usually concise and technical, naming
specific files/functions/line numbers, and calling out non-obvious reasoning
(gotchas, why an alternative was rejected, migrations involved, how
something was verified) rather than restating what the diff already shows.
Don't compress away detail a future session would need to avoid
re-deriving something the hard way. This is where the detail goes.

**2. Update `HANDOFF.md` where current-state facts changed.** Edit in place;
don't append a changelog here.
- A feature/page/module now works differently → update (or add) its
  description, short, with "details: history, session <date>".
- A new gotcha, workflow or convention → add it to the relevant section.
- "Known follow-ups" / TODO: remove what this session resolved, add what's
  newly deferred or discovered (with a pointer to the history section if
  there's more context there).
- Something stated here is no longer true → fix or delete it.
- Nothing current-state changed (e.g. a pure bug fix with no lasting
  gotcha) → leave `HANDOFF.md` alone; the history entry is enough.

Keep `HANDOFF.md` short. If it's creeping past a few hundred lines, move
detail into the history and leave a one-line summary and pointer behind.

## 6. Commit the handoff docs and push

- Check the tracking note (step 5) first. **Local-only**: leave both files
  out of `git add` entirely — they're still saved on disk with this
  session's update, just never staged. Consider adding them to `.gitignore`
  if they aren't already covered, so they don't get swept up by some other
  command later. **Committed to git**: stage `HANDOFF.md` and
  `HANDOFF-history.md` and commit per step 4's method.
- Push to the current branch's tracked upstream (`git push`), unless step 0
  established there's no remote and the user chose to skip pushing this
  session — in that case, just leave the commit local. (This push still
  happens even when the handoff docs are local-only, as long as there are
  other committed changes to push.)
- Report back: a short bullet list of what was committed, the commit
  hash(es), and confirmation the push succeeded (or an explicit note that it
  was skipped and why). Include a one-line mention of whether the handoff
  docs went into that push or were kept local-only, and which file(s) were
  updated (history entry added; `HANDOFF.md` updated or unchanged). If the
  project has a known auto-deploy (e.g. Vercel/Netlify watching this branch
  — check `HANDOFF.md`/`CLAUDE.md` for that), mention that it'll pick this
  up automatically; otherwise don't assume a deploy happens on push.

## Judgment calls

- Invoking this skill is itself the user's "push when ready" signal — don't
  ask for confirmation before committing/pushing just because a project's
  notes say "only push when explicitly asked." That said, for large or risky
  changes (migrations, anything touching money/auth/security), a one-line
  heads-up before pushing is still worth it.
- If unsure whether something belongs in the handoff at all, ask: "would a
  Claude with zero memory of this conversation need this to avoid
  re-discovering it, re-breaking it, or re-asking the user?" If yes, include
  it.
- If unsure which file it belongs in, ask: "does a new session need this by
  default, before it knows what it'll be working on?" Yes → `HANDOFF.md`
  (short), and the detail also goes in the history. No, it only matters once
  someone touches that area → `HANDOFF-history.md` only; they'll find it by
  grepping.
