# devvit-expert-skill

A Claude skill that makes Claude an expert Devvit developer. Drop it into any Cowork or Claude Code session and Claude instantly knows the full Devvit platform — useWebView, Redis, scheduler jobs, the Reddit API, message passing patterns, and more.

Built by [@chrismaz11](https://github.com/chrismaz11), founder of TrustSignal.

---

## What it does

Without this skill, Claude guesses at Devvit APIs and gets things wrong — hallucinating fake properties, misusing cron format, or missing the critical `event.data.data.message` double-nesting. With this skill, Claude writes correct Devvit code on the first try.

**Eval results:** 100% pass rate with skill vs 73% without (the key failure was a weekly scheduler job where Claude without the skill hallucinated `recurrence: 'weekly'` and `dayOfWeek: 'sunday'` — neither of which exist in Devvit).

---

## What's covered

- **useWebView** — full server↔web wiring, message passing, mount/unmount
- **Redis** — strings, hashes, sorted sets, lists, transactions, global scope
- **Scheduler** — cron jobs (ET timezone), one-time jobs, AppInstall pattern
- **Reddit API** — posts, comments, users, DMs, subreddit actions
- **Triggers** — AppInstall, PostSubmit, CommentCreate, ModAction, etc.
- **UIClient** — toasts, forms, navigation (Blocks context only)
- **Realtime** — pub/sub channels for live updates
- **13 copy-paste patterns** — leaderboard, user profiles, atomic counters, ET timezone, rate-limited DMs, error handling, and more
- **Key gotchas** — the double-nesting bug, ET cron, undefined userId, ui context limits

---

## Files

```
SKILL.md                  ← Main skill file (load this)
references/
  api.md                  ← Full API reference (@devvit/public-api 0.12.x)
  patterns.md             ← 13 copy-paste patterns
```

---

## How to install

### In Claude Cowork / Claude Code

1. Download or clone this repo
2. Copy the folder into your project's `.claude/skills/` directory:
   ```
   .claude/skills/devvit-expert/
     SKILL.md
     references/
       api.md
       patterns.md
   ```
3. Claude will automatically load it when you ask Devvit questions

### Manual (paste into context)

Copy the contents of `SKILL.md` into your system prompt or first message. Claude will read `references/api.md` and `references/patterns.md` on demand.

---

## Key rules the skill enforces

1. `event.data.data.message` — not `event.data.message` (two levels deep!)
2. Scheduler crons are ET (`'0 20 * * 0'` = Sunday 8 PM New York)
3. `context.userId` can be undefined — always null-check
4. `context.ui` is unavailable in scheduler jobs and triggers
5. All state goes in Redis — no SQL, no external DB by default

---

## r/devvit

If this saved you time, drop a comment on r/devvit or open an issue. PRs welcome.
