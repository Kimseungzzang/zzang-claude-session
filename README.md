# zzang-claude-session

Cross-session context storage for Claude Code — part of [zzang-claude-skills](https://github.com/Kimseungzzang/kimseungzzang-claude-skills).

Persists conversation context across sessions and machines so Claude can pick up exactly where you left off.

## How it works

`/session-save` compacts the current conversation into an ultra-dense format and pushes it to your private sessions repo. `/session-load` pulls it and orients Claude in seconds.

Each project gets its own folder. Context **accumulates** across sessions — decisions, failed attempts, and key facts are never lost.

## Structure

```
your-claude-sessions/
├── my-app/
│   ├── CURRENT.ctx        ← accumulated context (what /session-load reads)
│   ├── 2026-05-27T14-30   ← per-session snapshot (history)
│   └── 2026-05-28T09-15
└── another-project/
    ├── CURRENT.ctx
    └── ...
```

See the [`demo/`](./demo) folder for real examples.

## Setup

**1. Create your own private sessions repo** (one time):

```bash
gh repo create claude-sessions --private
```

**2. Install the skills:**

```bash
npx zzang-claude-skills
```

The installer will ask for your sessions repo URL and save it to `~/.claude/sessions-remote`.

**3. Use:**

```
# end of session
/session-save

# start of next session
/session-load
```

> On a new machine, just install the skills and run `/session-load` — it clones your sessions repo automatically.

## CURRENT.ctx format

~150–300 tokens. Designed to restore context fast without replaying the full conversation.

```
SESSION {timestamp} | {/absolute/cwd} | {branch}
STACK: {lang/framework/db}
DONE: {completed items — replaced each session}
CHANGED: {modified files — replaced each session}
TRIED: {failed attempts and why — accumulated, never compressed}
DECIDED: {decision: reasoning — accumulated, never compressed}
TODO: {pending tasks — completed items removed automatically}
OPEN: {unresolved questions — accumulated}
CTX: {non-obvious facts Claude must know — accumulated}
```

**Accumulation rules:**

| Field | Behavior |
|-------|----------|
| `TRIED` | Grows across sessions — prevents repeating the same mistakes |
| `DECIDED` | Grows across sessions — preserves the reasoning behind choices |
| `CTX`, `OPEN` | Grows across sessions |
| `DONE`, `CHANGED` | Replaced each session (history stays in snapshot files) |
| `TODO` | Completed items removed; new items added |

## Multiple projects in one session

`/session-save` detects all projects touched during the session, then saves **filtered context** to each project separately — no cross-contamination.
