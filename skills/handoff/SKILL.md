---
name: handoff
description: Saves and restores a project's continuity file (HANDOFF.md) so work can jump from one Claude session to the next without losing context — before a /clear, when the context window fills up, at the end of the day, or when switching between Claude Code, Cowork and Claude chat. Use it whenever the user says "/handoff", "handoff", "hand off", "save the context", "save state", "pick up where we left off", "resume", "where were we", "continue from yesterday", or the Spanish equivalents "guardá el contexto", "cortemos acá", "se me llena el contexto", "cerramos por hoy", "retomá el handoff", "seguí donde quedamos", "dónde quedamos", "retomemos". Also use it when a fresh session opens by naming a project the user was already working on, to look for its handoff before answering.
---

# Handoff Ledger

A Claude session dies with all of its working context inside it. This skill distills that context into one Markdown file per project that survives `/clear` or a closed window, and loads it back in the next session — on the same surface or a different one.

One word, `/handoff`, does both jobs. Decide which from the state of the session.

---

## Step 1 — Pick the mode

**LOAD** when the session is fresh: no files edited, no work commands run, nothing researched, the user hasn't described anything new. Usually within the first turn or two.

**SAVE** when there is work in the session: files touched, decisions made, something investigated, or the user briefed you on the state of something.

Tie-breakers:

- Already loaded a handoff in this session and invoked again → SAVE (the user is closing).
- `/handoff resume`, `/handoff load`, "where were we" → LOAD, always.
- Any other argument (`/handoff <text>`) → SAVE, and the text goes verbatim into `## User note`.
- Fresh session and no handoff exists for this project → say so and offer to start one, asking only for the goal and current state.

Don't ask "load or save?" unless you genuinely can't tell. A one-word command shouldn't charge a toll.

---

## Step 2 — Resolve where the file lives

**One project = one file, always the same one.** Never create `HANDOFF-v2.md`, `handoff-final.md` or similar: the whole value is that there is exactly one place to look. History lives in git or in the session log, not in extra files.

Walk this chain and keep **every** location that is available (it matters for mirroring):

**1. Project filesystem** — `<root>/HANDOFF.md`

Applies whenever you can write to the real project directory: Claude Code on a local checkout, or Cowork with a connected folder (use the device's shell there, not the cloud container — anything written in the container is destroyed with the session).

The root is the git top level if there is one (`git rev-parse --show-toplevel`), otherwise the working directory.

Never put it under `.claude/`: Claude Code treats that directory as protected, so every write there asks the user for approval and fails outright in non-interactive sessions. A handoff left at `<root>/.claude/HANDOFF.md` by an older version is still read on LOAD; on the next SAVE, write `<root>/HANDOFF.md` and tell the user in one line that the old file can be deleted (don't delete it yourself).

**2. Google Drive** — folder `Claude Handoffs/`, file `HANDOFF-<slug>.md`

Applies when there is no project filesystem but a Drive connector is available: typically Claude chat, or Cowork without a folder. It is the one place all three surfaces can reach, so it is the real bridge between chat and Code.

The `slug` is the project name lowercased, accents stripped, spaces to hyphens: `My Project` → `my-project`. It must be deterministic so a session on another surface finds the same file. Confirm it with the user the first time; after that it lives in the file's frontmatter and is not discussed again.

**3. Chat block** — last resort

With neither filesystem nor Drive, emit the file as a Markdown code block for the user to save by hand, and say in one line that connecting a folder or the Drive connector removes that manual step.

### Mirroring and divergence

Two copies that drift apart silently are worse than one inconvenient copy.

- **On save:** write to location 1 and, if Drive is also available, mirror the same content there. Confirm in one line where each copy ended up.
- **On load:** if both exist, compare the `updated` field in the frontmatter, use the newer one, and tell the user about the difference — they need to know a copy fell behind.

---

## Mode LOAD

1. **Read the whole file.** If it doesn't exist in any location, say so and offer SAVE.
2. **Check freshness before trusting it:**
   - If the file ends with a `Resumed:` line, another session already consumed it. Say so, don't act on its next step, and ask whether to resume anyway or write a new one.
   - If `updated` is more than 7 days old, say so up front: the state has probably moved.
3. **Verify against reality.** Run what `## How to verify` says, plus `git status --short` if there is a repo. If the disk contradicts the handoff (files it says were edited aren't, tests it calls green fail, the branch doesn't exist, a blocker no longer applies), say so before touching anything. A stale handoff taken at face value is worse than none.
4. **Summarize, don't dump.** You are rebuilding context for yourself, not making the user reread what they wrote. Reply in 5–8 lines: goal, where it stopped, what's next, what's blocked.
5. **Propose the first next step and wait for a go-ahead.** If the user's invocation already said to continue ("resume and keep going", "seguí"), and verification in step 3 passed, execute it directly.
6. **Stamp consumption.** Append one line `Resumed: <YYYY-MM-DD HH:mm>` at the end of the file with an edit, never a rewrite. It lets the next session tell a live handoff from a used one. Never delete the file: if the resume goes wrong, it's all that's left to try again.
7. **Optional, cosmetic:** if the host offers a tool to rename the current session (for example a session-title tool in the Claude desktop app), set the title to the handoff's `title` field. If no such tool exists or it fails, continue without comment.

Remember the path: the next `/handoff` in this session saves there.

---

## Mode SAVE

1. **Reread the conversation since the last change of topic.** Every item must trace back to something that actually happened in the session. Don't invent.
2. **Read the existing file if there is one** — you are updating, not starting over. A trailing `Resumed:` line is dropped on rewrite.
3. **Rewrite the state sections** with today's real state, add a new entry to the session log, and rotate it (see below).
4. **If you are overwriting an existing handoff, show the delta first**: 3–5 bullets like "next steps: closed 2, added 1" / "the auth blocker: resolved, removing it". Write only after the user's go-ahead. Overwriting a continuity file without showing what changed is the fastest way to lose the user's trust. A first-time file is written directly.
5. **Write with the file-writing tool**, not a shell heredoc: shells mangle backslashes in Windows paths and `$` in content.
6. **Close with three lines**, in the user's language: where each copy ended up, what it contains in one sentence, and that it's safe to `/clear` now — invoking this same skill again in the new session (with the exact command the user typed, e.g. `/handoff-ledger:handoff`) picks it up.

Never run `/clear` yourself: it's the user's command.

### Log rotation

The session log keeps the **3 most recent sessions**, 2–4 lines each.

Before dropping the oldest entry, check whether it says something that still matters — a decision, a discarded option, a live blocker. If so, promote it to the right section first, then drop it. The log records movement; it is not where information lives. A fact that survived three sessions stopped being an event and became state.

---

## File template

```markdown
---
project: <Readable name>
slug: <slug>
title: <project> — <current task, ≤6 words>
updated: <YYYY-MM-DD HH:mm ±hh:mm>
surface: <claude-code | cowork | chat>
locations:
  - <path or location of every known copy>
---

# HANDOFF — <Project name>

## 1. Goal and scope
<What is being built or fixed, and for whom, in 2–5 lines.>

**Out of scope:** <what was explicitly decided NOT to do>

## 2. State
- **Done:** <short, verifiable list; "tests green" only if they were actually run>
- **In progress:** <what was left half-done, with exact file and function>
- **Not started:** <pending work, blockers first>

## 3. Decisions (do not reopen)
| Decision | Discarded | Why |
|---|---|---|
| | | |

**Already tried and failed:**
- <what was tried, what error it gave, why it was dropped>

## 4. Environment and files
- **Repo / path:**
- **Branch:**
- **Files touched:** `<path>` — <what changed>
- **Bring the environment up:**
- **Known traps:** <hooks that block, env vars, flaky steps and how to dodge them>

## 5. How to verify
<Exact commands that prove the current state: test suite, build, smoke check.>

## 6. Next steps
1. [ ] <the first action, unambiguous: "Edit X lines A–B to …">
2. [ ] <…>

**Blockers:**
- <what is stuck, waiting on what and on whom>

## 7. Session log (last 3)
### <YYYY-MM-DD> — <surface>
- <what was done>

## User note
<Whatever the user passed as an argument, verbatim. Empty otherwise.>
```

`title` names the task, never the word "handoff" (e.g. `billing-api — retry on webhook timeout`).

---

## Content rules

**Never write secrets.** No tokens, API keys, connection strings, passwords, customer data under NDA, or identifiable incident details. This file may end up committed to a repo or synced to Drive; treat it as public. Record *where* a secret lives — "token in the local `.env`, variable `PAYMENTS_KEY`" — never its value. If the user dictated a credential during the session, don't transcribe it.

**The why is the payload.** "We use Postgres instead of SQLite" doesn't help the next session; "Postgres because we need concurrent writers from the worker" stops the debate from reopening next week. Every decision carries its reason. If you don't know the reason, ask instead of inventing one. The decisions section is the most valuable one in the file.

**Write for someone who wasn't there.** The session that reads this has none of your context. "Fix the parser thing" means nothing; "the parser in `ingest.py` breaks on timestamps without a timezone, see line 84" does.

**Concrete over complete.** Aim for 60–120 lines. A 400-line handoff nobody reads, and it eats the context it was meant to save. If a section doesn't apply on this surface — no repo in a chat session — drop it rather than filling it with "N/A".

**Separate what you know from what you assume.** If you are inferring the state of something rather than having verified it, mark it `(unverified)`. The next session will act on it.

**No pasted code**, except a snippet of ≤5 lines that would be impossible to reconstruct.

**Link, don't copy.** If there is a written plan or spec file, link it and say which task it's on instead of repeating it.

**Lessons learned don't belong here.** Durable rules ("always run X before Y in this repo") go to the project's `CLAUDE.md` or equivalent memory *before* cutting the session. The handoff is about this piece of work; it gets overwritten.

**Commit or ignore — ask once.** The first time you create `HANDOFF.md` in a repo, mention once whether it should be committed or added to `.gitignore`. Committed helps when the project is the user's or their team's; ignored when the repo belongs to someone else or the handoff holds personal notes. It's the user's call, made once per repo.
