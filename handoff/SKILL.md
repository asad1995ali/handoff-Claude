---
name: handoff
description: Wrap up a coding session in any git-backed project — commit and push all pending changes, then update that project's HANDOFF.md (at the repo root) with a dated summary so the next chat/session, which starts with zero memory of this one, has full context. Works in any directory/repo, not tied to one project. Use whenever the user says things like "wrap up", "let's push this", "commit and push", "deploy this", "end of session", "update the handoff doc", "let's call it there", or similar — in whatever project the chat is currently working in.
---

# Handoff: commit, push, and update HANDOFF.md

Every project loses conversation memory between chats. `HANDOFF.md` at the
repo root is the one artifact the *next* session reads to know what happened
— treat writing it as seriously as writing the code. A future Claude with
zero context about this conversation has to be able to pick up from it alone.

This skill is intentionally generic — it doesn't assume a stack, a build
tool, or a remote host. Discover the project's actual conventions each time
rather than assuming they match a previous project.

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
(commit locally, update HANDOFF.md) and say clearly in the final summary that
changes were committed but **not pushed**, since there's nowhere to push to.

**HANDOFF.md missing.** This one doesn't need a "should I create it?" ask —
just create it fresh at the repo root (see step 5). A missing HANDOFF.md
just means this is the first handoff for this project, not a problem to
flag. It does need **one** question the first time only: whether the file
itself should be committed to git (pushed like the rest of the repo) or kept
local-only (never staged, purely for this machine). See step 5 for exactly
how to ask this and where the answer gets recorded — check for that record
on every future invocation instead of asking again.

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
conversation tells you *why*, which matters for the HANDOFF.md write-up in
step 5.

If the tree is already clean, there's nothing to commit — only continue to
step 5 if the user still wants a HANDOFF.md note (e.g. a session that was
pure investigation, or made changes through an external system like a
database console that won't show up as a file diff).

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
- One commit for the session's work is usually fine. The HANDOFF.md update
  (step 6) can ride in the same commit or a trailing one — default to
  folding it in unless the session had several unrelated chunks of work
  worth separating in history. **Only fold it in if HANDOFF.md's own
  tracking note (see step 5) says it's committed to git** — if it's marked
  local-only, never `git add` it, regardless of what else is being
  committed this session.

## 5. Draft the HANDOFF.md section

Read the current `HANDOFF.md` at the repo root first. If it already exists,
skip straight to the "Insert a new section" step below — the
tracking-decision question (next paragraph) is only asked once, the first
time the file is created for a project.

**If it doesn't exist yet**, this is the first handoff for this project.
Before writing it, ask the user one question: should this `HANDOFF.md` be
committed to git and pushed like the rest of the repo, or kept **local-only**
(never staged/committed, purely a note to yourself on this machine)? Record
their answer directly in the file itself, near the top, e.g.:

```markdown
## Handoff tracking
This file is kept **local-only** — not committed to git. (decided <date>)
```
or
```markdown
## Handoff tracking
This file is **committed and pushed** to git along with the rest of the repo. (decided <date>)
```

This line is what makes the decision durable — on every future invocation of
this skill for this project, read it and follow it instead of asking again.
Only re-ask if the user brings it up themselves (e.g. they explicitly ask to
change how it's tracked) or the note is missing/ambiguous.

Then create the rest of the file: a short top section describing the
project, stack, and any workflow notes you've picked up this session, then
the dated section below. New content must match whatever voice the file
already has — most projects that use this pattern write it concise and
technical, naming specific files/functions/line numbers, and calling out
non-obvious reasoning (gotchas, why an alternative was rejected, migrations
involved) rather than restating what the diff already shows.

Insert a new section titled `## Changes — session <date>` (use today's
actual date; if a session already ran earlier today, suffix `(part N)`).
Place it in chronological order relative to any existing dated sections, and
before a "Known follow-ups"/"TODO"/"Next steps" section if one exists.

Structure per feature/fix:
```markdown
## Changes — session <date>

### <Feature or area name>
What changed, which files/functions, and any non-obvious reasoning.
```

Then:
- Update whatever "known follow-ups" / "TODO" section already exists: remove
  what this session resolved, add what's newly deferred or discovered.
- Update top-of-file sections (stack, conventions, "hard-won lessons") only
  if something this session actually changes those facts — don't touch them
  otherwise.

Don't rewrite the whole file, and don't compress away detail a future
session would need to avoid re-deriving something the hard way — match
whatever density of detail the file already has.

## 6. Commit HANDOFF.md and push

- Check HANDOFF.md's tracking note (step 5) first. **Local-only**: leave it
  out of `git add` entirely — it's still saved on disk with this session's
  update, just never staged. Consider adding it to `.gitignore` if it isn't
  already covered, so it doesn't get swept up by some other command later.
  **Committed to git**: stage and commit per step 4's method as usual.
- Push to the current branch's tracked upstream (`git push`), unless step 0
  established there's no remote and the user chose to skip pushing this
  session — in that case, just leave the commit local. (This push still
  happens even when HANDOFF.md itself is local-only, as long as there are
  other committed changes to push.)
- Report back: a short bullet list of what was committed, the commit
  hash(es), and confirmation the push succeeded (or an explicit note that it
  was skipped and why) — including a one-line mention of whether HANDOFF.md
  was included in that push or kept local-only. If the project has a known
  auto-deploy (e.g. Vercel/Netlify watching this branch — check
  HANDOFF.md/CLAUDE.md for that), mention that it'll pick this up
  automatically; otherwise don't assume a deploy happens on push.

## Judgment calls

- Invoking this skill is itself the user's "push when ready" signal — don't
  ask for confirmation before committing/pushing just because a project's
  notes say "only push when explicitly asked." That said, for large or risky
  changes (migrations, anything touching money/auth/security), a one-line
  heads-up before pushing is still worth it.
- If unsure whether something belongs in HANDOFF.md, ask: "would a Claude
  with zero memory of this conversation need this to avoid re-discovering
  it, re-breaking it, or re-asking the user?" If yes, include it.
