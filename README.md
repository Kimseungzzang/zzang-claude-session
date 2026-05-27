# zzang-claude-session

Cross-session context storage for Claude Code — part of [zzang-claude-skills](https://github.com/Kimseungzzang/kimseungzzang-claude-skills).

Persists conversation context across sessions and machines. Includes real-time tool-use logging so Claude can recover from mid-task interruptions (token exhaustion, app crashes) — not just clean session ends.

## Setup

**1. Install the skills** (handles everything automatically):

```bash
npx zzang-claude-skills
```

The installer will:
- Install `/session-save` and `/session-load` skills
- Install and register the `PostToolUse` hook (`task-log.sh`)
- Ask for your private sessions repo URL (or guide you to create one)

**2. Use:**

```
# end of session (or before token limit)
/session-save

# start of next session
/session-load
```

> On a new machine: `npx zzang-claude-skills` → `/session-load` — it clones your sessions repo automatically.

---

## How it works

### Overview

```
Every tool use  →  PostToolUse hook  →  task-log.md  (local only)
                                              │
                                    /session-save
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                    reads task-log     writes snapshot      updates CURRENT.ctx
                    (absorbs it)       (timestamped)        (accumulated state)
                          │
                          └──── git commit & push ──→  GitHub (private repo)
                                                               │
                                                       /session-load
                                                               │
                                              git pull → reads CURRENT.ctx
                                                       + checks task-log
                                                               │
                                                       orients you to resume
```

### `/session-save` flow

```
[1] Ensure ~/.claude/zzang-ctx is a git repo (clone if needed)
         │
[2] git pull --rebase
         │
[3] Detect all projects worked on this session
    └── git rev-parse --show-toplevel | xargs basename
    └── scan conversation for other project paths
    └── confirm list with user
         │
[4] For each project:
    ├─► read task-log.md  ← absorb into DONE/CHANGED, record last timestamp as TASK-LOG-ID
    └─► read CURRENT.ctx  ← load accumulated history
         │
[5] Write timestamped snapshot
    ~/.claude/zzang-ctx/{project}/YYYY-MM-DDTHH-MM
         │
[6] Merge into CURRENT.ctx  (see merge rules below)
         │
[7] Delete task-log.md  (absorbed — next task starts fresh)
         │
[8] git add . && git commit && git push
         │
[9] Report ✅
```

### `/session-load` flow

```
[1] Ensure ~/.claude/zzang-ctx is a git repo (clone if needed)
         │
[2] git pull --rebase
         │
[3] Detect project name
         │
[4] Read CURRENT.ctx
         │
[5] Compare task-log vs TASK-LOG-ID
         │
    ┌────┴─────────────────────────────────────────────────────┐
    │                                                          │
Case 1: IDs match               Case 2: task-log newer than SAVED_ID
→ Clean state                   → Unsaved work detected
→ Resume normally               → Show unsaved entries, ask to resume
    │                                          │
Case 3: task-log header > SESSION    Case 4: no local task-log
→ New task in progress               → Different machine
→ Show task-log as current context   → Use CURRENT.ctx only, inform user
    │                                          │
    └──────────────────┬───────────────────────┘
                       │
[6] Output brief orientation (≤150 words)
```

### PostToolUse hook — `task-log.sh`

Runs after every tool use. Appends one line to a local `task-log.md`:

```
## 2026-05-28T14:00 | my-app          ← created on first entry of the session
[14:01] Write      src/api/webhooks/stripe.ts
[14:03] Bash       npm test
[14:05] Edit       src/middleware/idempotency.ts
```

- **Local only** — never pushed directly to GitHub
- **Captured even on crash** — written before Claude responds, survives token exhaustion
- **Absorbed by `/session-save`** — merged into CURRENT.ctx, then deleted
- **Read by `/session-load`** — compared against `TASK-LOG-ID` to detect interrupted work

---

## Storage layout

```
~/.claude/
├── commands/
│   ├── session-save.md       ← skill definitions
│   └── session-load.md
├── scripts/
│   └── task-log.sh           ← PostToolUse hook
├── settings.json             ← hook registration
├── zzang-ctx-remote          ← your GitHub repo URL
└── zzang-ctx/                ← git repo (linked to GitHub)
    ├── my-app/
    │   ├── CURRENT.ctx       ← accumulated state  (pushed to GitHub)
    │   ├── task-log.md       ← live log           (local only, never pushed)
    │   └── 2026-05-28T14-00  ← timestamped snapshots (pushed)
    └── other-project/
        └── CURRENT.ctx
```

---

## CURRENT.ctx format

~150–300 tokens. Restores context fast without replaying the full conversation.

```
SESSION 2026-05-28T14:00 | /Users/kim/my-app | feat/payments
TASK-LOG-ID: 14:05
STACK: Next.js TypeScript PostgreSQL Stripe Redis
DONE: Stripe webhook handler; idempotency middleware; order status polling
CHANGED: src/api/webhooks/stripe.ts(new); src/middleware/idempotency.ts(new); src/lib/order.ts
TRIED: websocket for order status(mobile clients drop WS on background); Stripe webhook in edge runtime(crypto unavailable)
DECIDED: use polling not websocket: simpler, works on all clients; idempotency keys in Redis: TTL is native, avoids schema migration
TODO: handle dispute webhooks | add webhook replay UI | load test idempotency
OPEN: Stripe Radar vs custom fraud rules for high-value orders
CTX: STRIPE_WEBHOOK_SECRET set manually (not in .env.example); DB schema frozen until Q3; Redis TTL 24h per Stripe recommendation
```

**Merge rules:**

| Field | Behavior |
|-------|----------|
| `SESSION` | Always updated to latest timestamp |
| `TASK-LOG-ID` | Replaced with last absorbed task-log line time |
| `TRIED` | **Accumulates** — prevents repeating mistakes |
| `DECIDED` | **Accumulates** — preserves reasoning behind choices |
| `CTX`, `OPEN` | **Accumulates** |
| `DONE`, `CHANGED` | Replaced each session (history stays in snapshot files) |
| `TODO` | Completed items removed; new items added |

> `TRIED` and `DECIDED` are **never compressed** — they are the most valuable fields for continuity.

---

## Multi-machine workflow

```
Machine A                          Machine B
─────────                          ─────────
/session-save                      npx zzang-claude-skills
  → absorbs task-log                 → clones from zzang-ctx-remote
  → updates CURRENT.ctx            /session-load
  → pushes to GitHub                 → pulls CURRENT.ctx from GitHub
                                     → no task-log (Case 4 — expected)
                                     → resumes from saved state
```

---

## Multiple projects in one session

`/session-save` detects all projects touched during the session and writes **filtered context** to each project folder separately — no cross-contamination between projects.

---

## Demo

See [`demo/my-app/`](./demo/my-app/) for real example snapshots and a CURRENT.ctx.
