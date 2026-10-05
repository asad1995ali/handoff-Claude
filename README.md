# handoff — a Claude Code skill for session handoffs

**The problem:** every Claude Code session starts with total amnesia. Close
the chat, and everything about *why* you made a decision, what you tried and
discarded, and what's still half-finished is gone. The next session (or the
next teammate, or you in a week) has to re-derive all of it from a diff.

**handoff** is a [Claude Code](https://claude.com/claude-code) skill that
fixes this with one command at the end of a session: it commits and pushes
your pending changes, then updates two files at the repo root so the next
session can pick up cold with full context:

- **`HANDOFF.md`**: the *current state* of the project (stack, workflows,
  how each part works today, gotchas, known follow-ups). It stays short, and
  every new session reads it in full.
- **`HANDOFF-history.md`**: an append-only, dated log of what changed in
  each session and why. It's never read in full; sessions grep it for the
  feature or file they're about to touch.

The split keeps start-of-session reading cheap. A single file that holds every
session's log eventually grows to tens of thousands of tokens.

## What it does

When invoked, the skill:

1. Confirms there's a git repo and a remote to push to (asking first if
   either is missing or ambiguous — it never runs `git init` or adds a remote
   on its own).
2. Reviews `git status` / `git diff` and cross-references it against what
   actually happened in the conversation, so the write-up captures *why*,
   not just *what*.
3. Runs whatever verification step the project already has (`npm run
   build`/`test`/`lint`, etc.) — best effort, doesn't invent one.
4. Stages and commits the session's changes.
5. Appends a new `## Changes — session <date>` section to
   `HANDOFF-history.md`: what changed, which files/functions, and any
   non-obvious reasoning (gotchas, rejected alternatives, migrations). Then
   updates `HANDOFF.md` in place wherever a current-state fact changed,
   including the "known follow-ups" / TODO section. Projects with an older
   single-file `HANDOFF.md` are offered a one-time split.
6. Commits both files (unless you've asked to keep them local-only) and
   pushes everything.
7. Reports back exactly what was committed, the commit hash(es), and whether
   the push succeeded.

It's intentionally generic — no assumptions about stack, build tool, or
hosting provider. It reads each project's actual conventions instead of
imposing its own.

## Why not just rely on commit messages?

Commit messages describe one commit. `HANDOFF-history.md` accumulates a
running, dated narrative of a project across many sessions, and `HANDOFF.md`
distils it into what's true now — the kind of context a
new contributor (human or AI) would otherwise have to reconstruct by reading
the whole git log and guessing at intent.

## Installation

Claude Code skills live in a `skills/` directory as a folder containing a
`SKILL.md`. To install `handoff`:

**User-level (available in every project):**

```bash
git clone https://github.com/asad1995ali/handoff-Claude.git
cp -r handoff-Claude/handoff ~/.claude/skills/handoff
```

**Project-level (this repo only):**

```bash
git clone https://github.com/asad1995ali/handoff-Claude.git
cp -r handoff-Claude/handoff /path/to/your-project/.claude/skills/handoff
```

Restart Claude Code (or start a new session) and the skill will be picked up
automatically.

## Usage

Just talk to Claude Code naturally at the end of a session — no special
syntax required. Any of these will trigger it:

- "wrap up"
- "let's push this"
- "commit and push"
- "end of session"
- "update the handoff doc"
- "let's call it there"

You can also invoke it explicitly with `/handoff` if your setup supports
slash-command skill invocation.

## Example

An entry appended to `HANDOFF-history.md`:

```
## Changes — session 2026-06-30

### Auth token refresh
Fixed a race in `src/auth/refresh.ts` where two concurrent requests could
both trigger a refresh, invalidating each other's new token. Serialized
refresh behind a single in-flight promise instead of a mutex, since the
mutex approach deadlocked under Jest's fake timers (see rejected attempt in
this session's diff history).

### Known follow-ups
- Refresh promise isn't cleared if the request is aborted mid-flight —
  low priority, hasn't caused a real failure yet.
```

## License

MIT — see [LICENSE](LICENSE).
