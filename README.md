# Handoff Ledger

Long Claude sessions resend their whole context every turn. `/clear` fixes that, but it also wipes your working memory: what was decided, what was tried, what comes next.

**Handoff Ledger** gives you one command, `/handoff`, that does both halves:

- **At the end of a session** it distills the work into a single `HANDOFF.md` per project: goal, state, decisions *with their reasons*, what already failed, how to verify, and the exact next step.
- **At the start of the next one** it reads that file back, checks it against the real repo (`git status`, the verify commands), summarizes it in a few lines and proposes the next step.

It works in Claude Code, Cowork and Claude chat. Where there's no filesystem, it falls back to Google Drive (if connected) or a copy-paste block.

## Install

```bash
claude plugin marketplace add PapiScholz/handoff-ledger
```

```bash
claude plugin install handoff-ledger@handoff-ledger
```

## Use

| You say | It does |
|---|---|
| `/handoff-ledger:handoff` after doing work | Writes or updates `<repo>/HANDOFF.md`, shows what changed, tells you it's safe to `/clear` |
| `/handoff-ledger:handoff` in a fresh session | Loads the handoff, verifies it against disk, summarizes, proposes the next step |
| `/handoff-ledger:handoff resume` | Always loads |
| `/handoff-ledger:handoff <note>` | Saves, keeping your note verbatim |

You don't have to type the command: phrases like "save the context", "where were we" or "pick up where we left off" (and their Spanish equivalents) trigger it too.

## What the file looks like

One file per project, always the same path, overwritten on every save. It contains:

1. Goal and scope (including what's explicitly out of scope)
2. State: done / in progress / not started
3. Decisions with the reason behind each one, plus what was tried and failed
4. Environment and files touched
5. How to verify the current state
6. Next steps and blockers
7. A log of the last 3 sessions

Safeguards built in:

- **No secrets**: it records *where* a credential lives, never its value.
- **Stale detection**: warns when a handoff is more than 7 days old or was already resumed by another session.
- **Verify before trust**: if the disk contradicts the handoff, it says so before touching anything.
- **Shows the delta** before overwriting an existing handoff.

The first time it creates `HANDOFF.md` in a repo, it asks whether you want it committed or git-ignored. It lives at the repo root rather than under `.claude/` because Claude Code asks for approval on every write to `.claude/`; handoffs left there by version 1.0.0 are still picked up.

## License

MIT
